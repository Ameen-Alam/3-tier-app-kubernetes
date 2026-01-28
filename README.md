# Kubernetes Course (5 Classes × 3 Hours) — Teacher Runbook

## Global Setup (Do this once, at start of Class 1)

### Verify tooling

```bash
kubectl version --client
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns
```

### Optional convenience aliases (recommended)

```bash
alias k=kubectl
export do="--dry-run=client -o yaml"
```

### Standard cleanup command (use after each lab)

```bash
k get all -A
```

---

# CLASS 1 (3 Hours): Kubernetes Mental Model + Pods + kubectl Debugging

## Prerequisite recall (Review 15 min before class)

* Docker: image vs container, ports, env vars
* YAML basics: indentation, lists/maps
* REST API concepts
* Linux: processes + exit codes

---

## 0:00–0:25 — Mental model

### Teach (talk track)

* Kubernetes is an **API + desired state** system.
* You declare objects; controllers reconcile reality.
* `metadata/spec/status` pattern:

  * **spec** = desired
  * **status** = actual
* Pod is the **smallest schedulable unit**, not container.

### Demo: API objects exist in etcd and surface via API

```bash
k api-resources | head
k explain pod | head
k explain pod.spec | head
```

---

## 0:25–1:20 — Pods, Namespaces, kubectl fundamentals (hands-on)

### Lab 1: Create namespace + nginx pod

```bash
k create ns mynamespace
k run nginx --image=nginx --restart=Never -n mynamespace
k get pod -n mynamespace -o wide
```

### Lab 2: Generate YAML without creating + apply

```bash
k run nginx --image=nginx --restart=Never -n mynamespace $do > nginx-pod.yaml
cat nginx-pod.yaml
k delete pod nginx -n mynamespace
k apply -f nginx-pod.yaml
k get pod -n mynamespace
```

### Lab 3: Logs + Exec

```bash
k logs -n mynamespace nginx
k exec -it -n mynamespace nginx -- /bin/sh
# inside:
ls
exit
```

---

## 1:20–1:50 — Debugging patterns (events are your truth)

### Create a broken pod on purpose

```bash
k run bad --image=nginx:9.99 --restart=Never -n mynamespace
k get pod -n mynamespace
k describe pod -n mynamespace bad
```

**Must highlight**

* Look at **Events** at the bottom: `ErrImagePull`, `ImagePullBackOff`.

Cleanup:

```bash
k delete pod -n mynamespace bad
```

---

## 1:50–2:30 — Busybox “run once” patterns (CKAD style)

### Lab: Print env and auto-delete

```bash
k run bb --image=busybox --restart=Never -it --rm -- env
```

### Lab: Echo hello world then exit

```bash
k run bb --image=busybox --restart=Never -it --rm -- /bin/sh -c "echo hello world"
```

### Lab: Create a pod with env var and verify

```bash
k run nginxenv --image=nginx --restart=Never --env=var1=val1 -n mynamespace
k exec -it -n mynamespace nginxenv -- sh -c 'echo $var1'
k delete pod -n mynamespace nginxenv
```

---

## 2:30–2:55 — Quick “Pod IP test” (foundation for networking)

```bash
k run nginx80 --image=nginx --restart=Never --port=80 -n mynamespace
NGINX_IP=$(k get pod nginx80 -n mynamespace -o jsonpath='{.status.podIP}')
k run bb --image=busybox --restart=Never -it --rm -n mynamespace -- \
  sh -c "wget -qO- ${NGINX_IP}:80"
k delete pod -n mynamespace nginx80
```

---

## 2:55–3:00 — Wrap & Homework briefing

### Homework (self-learning)

* Read official docs: Pods, Namespaces, kubectl basics
* Practice:

  1. Create pod YAML using dry-run
  2. Use `describe` to identify failure reason
  3. Use `logs -p` for previous container (simulate with bad image change)

### Self-assessment checklist

* Can you explain `spec vs status` in 2 minutes?
* Can you debug `ImagePullBackOff` and point to the event line?

### Cleanup

```bash
k delete ns mynamespace
```

---

# CLASS 2 (3 Hours): Pod Design + Multi-container + Workloads (Deployments/Jobs/CronJobs)

## Prerequisite recall

* Stateless architecture
* Rolling deployment ideas (blue/green/canary concept)
* Linux process exit codes

---

## 0:00–0:40 — Multi-container Pods (sidecar + init)

### Lab 1: Two busybox containers share emptyDir

Create `multi.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi
spec:
  restartPolicy: Never
  volumes:
  - name: shared
    emptyDir: {}
  containers:
  - name: bb1
    image: busybox
    command: ["/bin/sh","-c","sleep 3600"]
    volumeMounts:
    - name: shared
      mountPath: /data
  - name: bb2
    image: busybox
    command: ["/bin/sh","-c","sleep 3600"]
    volumeMounts:
    - name: shared
      mountPath: /data
```

Apply + test:

```bash
k apply -f multi.yaml
k exec -it multi -c bb2 -- sh -c "echo hi-from-bb2 > /data/msg"
k exec -it multi -c bb1 -- sh -c "cat /data/msg"
k delete pod multi
```

### Lab 2: Init container writes HTML for nginx

Create `init-nginx.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-nginx
spec:
  volumes:
  - name: web
    emptyDir: {}
  initContainers:
  - name: init
    image: busybox
    command: ["/bin/sh","-c","echo 'Test' > /work-dir/index.html"]
    volumeMounts:
    - name: web
      mountPath: /work-dir
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: web
      mountPath: /usr/share/nginx/html
```

Apply + test:

```bash
k apply -f init-nginx.yaml
IP=$(k get pod init-nginx -o jsonpath='{.status.podIP}')
k run bb --rm -it --restart=Never --image=busybox -- wget -qO- http://$IP
k delete pod init-nginx
```

---

## 0:40–1:50 — Deployments: scale, rollout, rollback

### Lab: Create deployment 2 replicas, expose containerPort 80

Fast path:

```bash
k create deploy web --image=nginx:1.18.0
k scale deploy web --replicas=2
k get deploy web
k get rs
k get pods -l app=web
```

Add port via edit (or apply YAML). Teacher method:

```bash
k edit deploy web
```

Inside editor, under container:

```yaml
ports:
- containerPort: 80
```

Rollout check:

```bash
k rollout status deploy web
k get pods -l app=web -o wide
```

### Update image + rollback

```bash
k set image deploy/web nginx=nginx:1.19.8
k rollout status deploy web
k rollout history deploy web
k rollout undo deploy web
k rollout status deploy web
```

### Force a failure image (debug rollout)

```bash
k set image deploy/web nginx=nginx:9.99
k rollout status deploy web
k get pods
k describe pod <one-failing-pod>
```

Fix to known good:

```bash
k set image deploy/web nginx=nginx:1.19.8
k rollout status deploy web
```

---

## 1:50–2:30 — Jobs

### Lab: Job that echoes hello then world

```bash
k create job hello --image=busybox -- /bin/sh -c "echo hello;sleep 5;echo world"
k wait --for=condition=complete --timeout=120s job/hello
k logs job/hello
k delete job hello
```

### Lab: Job completions=5 sequential

```bash
k create job seq --image=busybox $do -- /bin/sh -c "echo run; sleep 2" > job.yaml
# edit job.yaml: add spec.completions: 5
# also set restartPolicy: OnFailure remains OK
k apply -f job.yaml
k get job seq -w
k delete job seq
```

---

## 2:30–2:55 — CronJobs

```bash
k create cronjob cj --image=busybox --schedule="*/1 * * * *" -- /bin/sh -c "date; echo Hello"
k get cj
k get jobs -w
k get pods --show-labels | grep cj
k logs <pod-created-by-cronjob>
k delete cj cj
```

---

## Homework

* Practice writing a Deployment YAML from scratch (no generator)
* Run 1 CronJob and confirm job/pod mapping via labels
* Self-test: explain difference between Deployment vs Job vs CronJob

## Cleanup

```bash
k delete deploy web
```

---

# CLASS 3 (3 Hours): Config, Security, Resource Control (ConfigMaps/Secrets/ServiceAccounts/Requests)

## Prerequisite recall

* Env vars, config patterns (12-factor)
* Secrets handling principles
* Linux users/UIDs

---

## 0:00–1:10 — ConfigMaps (env + volume)

### Create ConfigMap literals

```bash
k create cm config --from-literal=foo=lala --from-literal=foo2=lolo
k get cm config -o yaml
```

### ConfigMap from file

```bash
echo -e "foo3=lili\nfoo4=lele" > config.txt
k create cm configmap2 --from-file=config.txt
k describe cm configmap2
```

### ConfigMap into env var using key

```bash
k create cm options --from-literal=var5=val5
cat > cm-env.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: cm-env
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: option
      valueFrom:
        configMapKeyRef:
          name: options
          key: var5
YAML
k apply -f cm-env.yaml
k exec -it cm-env -- sh -c 'echo $option'
k delete pod cm-env
```

### ConfigMap as volume

```bash
k create cm cmvolume --from-literal=var8=val8 --from-literal=var9=val9
cat > cm-vol.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: cm-vol
spec:
  volumes:
  - name: v
    configMap:
      name: cmvolume
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: v
      mountPath: /etc/lala
YAML
k apply -f cm-vol.yaml
k exec -it cm-vol -- sh -c 'ls /etc/lala && cat /etc/lala/var8'
k delete pod cm-vol
```

---

## 1:10–2:05 — Secrets (env + volume)

### Secret literal

```bash
k create secret generic mysecret --from-literal=password=mypass
k get secret mysecret -o yaml
```

### Secret from file + decode

```bash
echo -n admin > username
k create secret generic mysecret2 --from-file=username
k get secret mysecret2 -o jsonpath='{.data.username}' | base64 -d; echo
```

### Mount secret as volume

```bash
cat > secret-vol.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: sec-vol
spec:
  volumes:
  - name: s
    secret:
      secretName: mysecret2
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: s
      mountPath: /etc/foo
      readOnly: true
YAML
k apply -f secret-vol.yaml
k exec -it sec-vol -- sh -c 'ls /etc/foo && cat /etc/foo/username'
k delete pod sec-vol
```

### Secret into env var

```bash
cat > secret-env.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: sec-env
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret2
          key: username
YAML
k apply -f secret-env.yaml
k exec -it sec-env -- sh -c 'echo $USERNAME'
k delete pod sec-env
```

---

## 2:05–2:35 — SecurityContext + ServiceAccounts

### SecurityContext runAsUser

```bash
cat > sc.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: sc
spec:
  securityContext:
    runAsUser: 101
  containers:
  - name: nginx
    image: nginx
YAML
# no need to apply in exam sometimes, but here we can:
k apply -f sc.yaml
k delete pod sc
```

### ServiceAccount + token check

```bash
k create sa myuser
cat > sa-pod.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: sa-pod
spec:
  serviceAccountName: myuser
  containers:
  - name: nginx
    image: nginx
    command: ["/bin/sh","-c","sleep 3600"]
YAML
k apply -f sa-pod.yaml
k exec -it sa-pod -- sh -c 'ls /var/run/secrets/kubernetes.io/serviceaccount || true'
k delete pod sa-pod
```

---

## 2:35–2:55 — Requests/Limits + Quotas (quick demo)

### Requests/limits pod

```bash
cat > res.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: res
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "200m"
        memory: "512Mi"
YAML
k apply -f res.yaml
k describe pod res | grep -A5 -i "requests"
k delete pod res
```

---

## Homework

* Build one pod that uses:

  * ConfigMap via volume
  * Secret via env
  * runAsUser securityContext
  * requests/limits
* Self-test: explain difference `env` vs `envFrom` and `configMapKeyRef` vs `configMapRef`.

## Cleanup

```bash
k delete cm config configmap2 options cmvolume --ignore-not-found
k delete secret mysecret mysecret2 --ignore-not-found
k delete sa myuser --ignore-not-found
rm -f username config.txt
```

---

# CLASS 4 (3 Hours): Services + NetworkPolicy + Storage (PV/PVC) + Troubleshooting

## Prerequisite recall

* TCP/IP, ports, DNS
* Stateless vs stateful
* Filesystems basics

---

## 0:00–1:20 — Services (ClusterIP, NodePort) + Endpoints

### Pod + ClusterIP service via `--expose`

```bash
k run nginx --image=nginx --restart=Never --port=80 --expose
k get svc nginx
k get ep nginx
```

Test ClusterIP:

```bash
IP=$(k get svc nginx -o jsonpath='{.spec.clusterIP}')
k run bb --rm -it --restart=Never --image=busybox -- wget -qO- $IP:80 --timeout=2
```

Convert to NodePort:

```bash
k patch svc nginx -p '{"spec":{"type":"NodePort"}}'
k get svc nginx
NP=$(k get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
echo "NodePort=$NP"
```

Teacher note (environment dependent):

* minikube: `minikube ip`
* kind/docker-desktop: often `127.0.0.1`

Cleanup:

```bash
k delete svc nginx
k delete pod nginx
```

---

## 1:20–2:05 — NetworkPolicy (label-based access)

Create deployment + service:

```bash
k create deploy web --image=nginx --replicas=2
k expose deploy web --port=80
```

Apply NetworkPolicy:

```bash
cat > netpol.yaml <<'YAML'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-from-granted
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: granted
YAML
k apply -f netpol.yaml
```

Test (may vary if cluster enforces netpol):

```bash
k run bb --rm -it --restart=Never --image=busybox -- wget -qO- http://web:80 --timeout=2
k run bb --rm -it --restart=Never --image=busybox --labels=access=granted -- wget -qO- http://web:80 --timeout=2
```

Teacher must explain:

* If both succeed → CNI doesn’t enforce NetworkPolicy.

---

## 2:05–2:50 — Storage: emptyDir vs PV/PVC (hostPath lab)

### PV

```bash
cat > pv.yaml <<'YAML'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: myvolume
spec:
  storageClassName: normal
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  hostPath:
    path: /etc/foo
YAML
k apply -f pv.yaml
k get pv
```

### PVC

```bash
cat > pvc.yaml <<'YAML'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  storageClassName: normal
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
YAML
k apply -f pvc.yaml
k get pvc
k get pv
```

### Pod mounts PVC + writes file

```bash
cat > pvc-pod.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  restartPolicy: Never
  volumes:
  - name: myvolume
    persistentVolumeClaim:
      claimName: mypvc
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh","-c","sleep 3600"]
    volumeMounts:
    - name: myvolume
      mountPath: /etc/foo
YAML
k apply -f pvc-pod.yaml
k exec -it busybox -- cp /etc/passwd /etc/foo/passwd
k exec -it busybox -- ls -l /etc/foo
```

### Second pod reads file (demonstrate node-local caveat)

```bash
sed 's/name: busybox/name: busybox2/' pvc-pod.yaml > pvc-pod2.yaml
k apply -f pvc-pod2.yaml
k exec -it busybox2 -- ls -l /etc/foo
```

If file missing, show why:

```bash
k get pod busybox -o wide
k get pod busybox2 -o wide
```

---

## Homework

* Build service for a deployment and hit via DNS name
* Write a NetworkPolicy and explain expected behavior
* Storage: explain why hostPath fails cross-node and what storage solves it (NFS/CSI)

## Cleanup

```bash
k delete netpol allow-web-from-granted --ignore-not-found
k delete svc web --ignore-not-found
k delete deploy web --ignore-not-found
k delete pod busybox busybox2 --ignore-not-found
k delete pvc mypvc --ignore-not-found
k delete pv myvolume --ignore-not-found
rm -f pv.yaml pvc.yaml pvc-pod.yaml pvc-pod2.yaml netpol.yaml
```

---

# CLASS 5 (3 Hours): Helm + CRDs + “Production Thinking” Capstone Kickoff

## Prerequisite recall

* YAML fluency
* Versioning idea (v1/v2)
* Packaging/deployment habits (like Docker/Compose)

---

## 0:00–1:20 — Helm essentials (install/upgrade/rollback/values)

### Verify helm

```bash
helm version
```

### Add repo + update

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
```

### Inspect chart values

```bash
helm show values bitnami/node | head -n 40
helm show values bitnami/node | grep -i replica
```

### Install with replicas=2 (fast test)

```bash
helm install mynode bitnami/node --set replicaCount=2
kubectl get deploy
kubectl get pods
```

### Upgrade to replicas=5

```bash
helm upgrade mynode bitnami/node --set replicaCount=5
kubectl get deploy
kubectl get pods
```

### Rollback

```bash
helm history mynode
helm rollback mynode 1
kubectl get deploy
```

### Uninstall

```bash
helm uninstall mynode
```

### Helm list pending all namespaces (demo)

```bash
helm list -A --pending
```

---

## 1:20–2:20 — CRDs (create CRD + create custom resource + list)

### Create CRD manifest `operator-crd.yaml`

```bash
cat > operator-crd.yaml <<'YAML'
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: operators.stable.example.com
spec:
  group: stable.example.com
  scope: Namespaced
  names:
    plural: operators
    singular: operator
    kind: Operator
    shortNames:
    - op
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              email:
                type: string
              name:
                type: string
              age:
                type: integer
YAML
```

Apply + verify:

```bash
kubectl apply -f operator-crd.yaml
kubectl get crd | grep operators
kubectl describe crd operators.stable.example.com | head
```

### Create a custom resource `operator.yaml`

```bash
cat > operator.yaml <<'YAML'
apiVersion: stable.example.com/v1
kind: Operator
metadata:
  name: operator-sample
spec:
  email: operator-sample@stable.example.com
  name: "operator sample"
  age: 30
YAML
kubectl apply -f operator.yaml
```

List:

```bash
kubectl get operators
kubectl get operator
kubectl get op
kubectl get operator operator-sample -o yaml
```

Cleanup CRD demo:

```bash
kubectl delete -f operator.yaml
kubectl delete -f operator-crd.yaml
```

Teacher note:

* CRD adds a **new API resource**, but without a controller it won’t “do” anything—CKAD focuses on creation/usage.

---

## 2:20–3:00 — Capstone briefing (students do at home)

### Capstone Project (Homework for Week 5)

**Goal:** “Deploy an AI-Agent style service on Kubernetes” (simulation OK)

Must include:

1. A Deployment + Service
2. ConfigMap + Secret
3. Resource requests/limits
4. NetworkPolicy restricting ingress to label `access=granted`
5. PVC mounted at `/var/app` (or `/data`)
6. Helm chart packaging for your app OR use Helm to deploy a dependency (like redis)
7. A CRD + one Custom Resource (used as config spec, even if no controller)

Deliverables:

* `k8s/` folder with YAMLs
* `helm/` chart folder (if chart used)
* `README.md` explaining:

  * architecture
  * how to deploy
  * how to test
  * common failure modes + how to debug
