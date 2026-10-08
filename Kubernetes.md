# Kubernetes — Explained Like a Novice

> A complete beginner's guide: what Kubernetes is, why it exists, every core concept in plain language, and what role it plays in Spring Boot cloud native development.

---

## 1. The Problem: Why Kubernetes Exists

### The story

**Stage 1 — One app, one server.** You build a Spring Boot app, run it on a server, done. Life is simple.

**Stage 2 — Containers arrive.** Docker lets you package your app + everything it needs into a portable box (a "container image") that runs identically anywhere. Great — but now companies don't run one app; they run dozens of microservices, each needing several copies for load and reliability.

**Stage 3 — Chaos.** Now you have **200 containers that need to live across 40 servers**, and every question is a nightmare:

- Which server should each container run on? Who tracks free capacity?
- A container crashes at 3 AM — who restarts it? A whole *server* dies — who moves its containers elsewhere?
- Traffic doubles during a sale — who starts more copies, and stops them after?
- Containers keep dying and being recreated on different machines with different IPs — how does Service A find Service B?
- How do you update 50 running copies to a new version **without downtime**?
- Where do settings and passwords live, when containers are disposable?

Humans doing this manually don't scale. You need software that manages containers the way an operating system manages programs on one machine — but across a whole fleet.

**Kubernetes is that software.** Its name comes from Greek: *kubernētēs* = "helmsman/pilot" — the one who steers the ship. (Also called **K8s** — "K" + 8 letters + "s".)

### The shipping port analogy

- **Containers** = standardized shipping containers (this is literally where the metaphor comes from)
- **Servers** = ships in a fleet
- **Kubernetes** = the port authority that decides which ship carries which containers, notices when one falls overboard, replaces it, and keeps the manifest up to date

You never deal with individual ships — you tell the port authority what cargo you have, and it handles placement, losses, and routing.

### Wait — why does only ONE container crash? Shouldn't a bug kill all replicas?

Good intuition — but here's the key: **replicas share the same code, not the same life.** Each pod has its own memory, sits on its own node, and receives its own stream of requests. Same blueprint, different experiences.

**Analogy**: identical cars from the same factory — same design, same parts. But only the one that hit the pothole breaks down.

Most real crashes are **non-deterministic** — they depend on what *that particular pod* ran into:

| Cause | Why it kills just one pod |
|---|---|
| **Poison request** | 1,000 normal requests spread across replicas; one malformed/huge request lands on replica #2 and kills only it |
| **Memory pressure** | Each pod's memory grows differently; replica #1 crossed its limit first → `OOMKilled`, others haven't yet |
| **Node problems** | That pod's machine had disk pressure, a hardware fault, or a "noisy neighbor" eating resources |
| **Race conditions/timing** | A rare thread interleaving triggered only under that pod's exact load at that moment |
| **External blips** | A dependency timeout or network hiccup hit that pod's connection, not others' |

**BUT — the other case is real too.** If the failure is *deterministic* — bad config, can't reach the DB at startup, a null pointer on every request — then **all replicas crash together**: `CrashLoopBackOff` across the whole Deployment.

This distinction defines incident severity:

- **One pod crashed** → routine. Kubernetes restarts it, life goes on. This is exactly *why* replicas exist — redundancy assumes failures are partial and random.
- **All pods crashlooping** → the deployment itself is broken (bad image, bad config, dead dependency). Wake the team — self-healing can't fix a bug that's in every copy.

---

## 2. The One Big Idea: Desired State

**You declare what you want. Kubernetes makes it true — forever.**

```yaml
# "I want 3 copies of my-app running at all times"
replicas: 3
image: my-app:1.0
```

You don't say "start container on server 7, then monitor it, then restart it if it fails." You say **"3 replicas"**, and Kubernetes controllers run an endless loop:

```
LOOP FOREVER:
    look at actual state
    compare to desired state
    if different → fix it
```

A container dies? Loop notices "2 running, wanted 3" → starts a new one. A whole server dies? Loop reschedules all its containers onto healthy servers. This is called the **reconciliation loop**, and it explains almost everything about Kubernetes:

- Why config is **declarative YAML** (you describe the goal, not the steps)
- Why it **self-heals** (the loop never stops checking)
- Why create/update are the same command (`kubectl apply` — "here's the new desired state")

**Analogy: a thermostat.** You don't tell it "turn heater on for 4 minutes." You set 22°C and it does whatever it takes, continuously.

---

## 3. Architecture: The City and Its Managers

A Kubernetes **cluster** has two halves:

### Control Plane — "the management office"
The brain. Runs the components that make decisions:

| Component | Plain-language job |
|---|---|
| **API Server** | The front desk. EVERYTHING talks to it — you, other components. The only door in. |
| **etcd** | The filing cabinet. A database storing the entire desired state of the cluster. |
| **Scheduler** | The dispatcher. Decides *which* server each new pod runs on (based on capacity, rules). |
| **Controller Manager** | The supervisors. Runs the reconciliation loops that keep reality matching desired state. |

### Nodes (Worker Nodes) — "the workers"
The machines (VMs or physical servers) where containers actually run. Each node has:

- **kubelet** — the node's personal manager; takes orders from the control plane, starts/stops containers, reports health
- **container runtime** — actually runs the containers (containerd)
- **kube-proxy** — handles networking so traffic reaches the right containers

### Flow example
`kubectl apply -f deployment.yaml` → API server stores it in etcd → scheduler picks nodes → kubelets pull the image and start containers → controllers keep watching forever.

---

## 4. Core Objects — Every Concept in Plain Language

### 4.1 Pod — the atom

**The smallest unit Kubernetes manages.** A wrapper around one (rarely more) container.

- Why not run containers directly? The pod adds what containers lack: its own IP address, shared storage, lifecycle rules.
- **Pods are disposable.** They die and get recreated constantly — with *different IPs*. Never hardcode or trust a pod's identity.
- Analogy: not the cargo itself, but the standardized shipping container it travels in — crane operators only handle containers.

### 4.2 Deployment — the replica manager

**"Keep N identical pods running, and let me update them safely."**

- You declare `replicas: 3` + an image; it keeps 3 pods alive, always.
- Handles **rolling updates** — swaps old pods for new ones gradually, a few at a time → zero downtime.
- Handles **rollbacks** — bad release? One command returns to the previous version.
- Internally uses a **ReplicaSet** (know it exists; you rarely touch it directly).

### 4.3 Service — the stable address

Pods die and get new IPs constantly. A **Service** is a stable virtual IP + DNS name that load-balances across whichever matching pods are currently alive.

- `payment-service` is always reachable at that name — Kubernetes routes to healthy pods behind it.
- Types: **ClusterIP** (internal only, default), **NodePort** (expose on each node), **LoadBalancer** (gets a real cloud LB).

**This is how services find each other** — by service name/DNS, never by pod IP.

### 4.4 Labels & Selectors — name tags

Key-value tags you stick on pods (`app=payment`, `version=v2`). Everything else finds pods *through* labels:

- A Service routes to pods with `app=payment`
- A Deployment manages pods with its label

Analogy: airport luggage tags. The system doesn't know your bag personally — it matches tags.

### 4.5 Namespace — folders

Logical partitions inside one cluster (`dev`, `staging`, `prod`, or per-team). Same service name can exist in two namespaces without collision.

### 4.6 ConfigMap & Secret — settings and passwords

- **ConfigMap** — non-sensitive config (URLs, feature flags) injected as env vars or files. Lets the same image run differently per environment — a core 12-factor requirement.
- **Secret** — same mechanism for sensitive values (passwords, tokens), stored base64. (For real security, pair with Vault or cloud secret managers.)

### 4.7 Probes — health checks

Kubernetes asking your app "are you okay?" — three different questions:

| Probe | Question it asks | If it fails |
|---|---|---|
| **Liveness** | "Are you alive or deadlocked?" | K8s **restarts** the container |
| **Readiness** | "Can you take traffic right now?" | Pod **removed from Service** endpoints (not restarted) |
| **Startup** | "Have you finished booting?" | Delays the other two probes |

> **Java relevance**: Spring Boot apps start slowly. Without a **startup probe**, a liveness probe may kill your app during boot — it restarts, gets killed again, forever. Startup probes exist largely because of JVM apps.

### 4.8 Resources — requests & limits

- **requests** = reservation — "guarantee this pod at least X CPU / Y memory." The scheduler uses it to pick a node.
- **limits** = the cap — exceed memory limit → **OOMKilled** (pod killed); exceed CPU limit → **throttled** (slowed, not killed).
- Quality-of-service tiers result: **Guaranteed** (requests=limits) > **Burstable** > **BestEffort** (no requests — first to be evicted under pressure).

### 4.9 HPA — automatic staffing

**Horizontal Pod Autoscaler**: watches metrics (CPU%, memory, or custom) and changes replica count automatically.

- Sale traffic doubles → HPA scales 3 pods → 10 → back down after. No human involved.

### 4.10 Storage — PV / PVC

Containers' filesystems are disposable. For data that must survive (databases, Kafka):

- **PersistentVolume (PV)** — a piece of real storage (disk, cloud volume)
- **PersistentVolumeClaim (PVC)** — a pod's *request* for storage, like filing a requisition form

### 4.11 The other workload types — know what they're FOR

| Object | One-line purpose |
|---|---|
| **StatefulSet** | Pods needing stable identity/storage — databases, Kafka, ZooKeeper |
| **DaemonSet** | Exactly one pod per node — log collectors, monitoring agents |
| **Job** | Run once to completion — migrations, batch tasks |
| **CronJob** | Jobs on a schedule — nightly reports |
| **Ingress** | HTTP front door — routes external traffic by host/path to Services |
| **NetworkPolicy** | Firewall rules for pod-to-pod traffic |
| **RBAC** | Who may do what — Roles, RoleBindings, ServiceAccounts |

### 4.12 kubectl — the remote control

The CLI you talk to the cluster with. The five commands you'll use daily:

```bash
kubectl get pods            # what's running?
kubectl describe pod <p>    # WHY is it broken — read the Events section
kubectl logs <p> --previous # last words of a crashed container
kubectl exec -it <p> -- sh  # go inside
kubectl apply -f file.yaml  # declare desired state
```

---

## 5. A Request's Journey (End-to-End Example)

You deploy `payment-api` (3 replicas) behind a Service + Ingress:

1. `kubectl apply` your Deployment → API server → etcd → scheduler places 3 pods on nodes with free capacity
2. Pods boot; **startup probe** gives the JVM time; once up, **readiness probe** passes → pod added to Service endpoints
3. User hits `https://api.example.com/pay` → Ingress → `payment-api` Service → load-balanced to a healthy pod
4. Traffic spikes → **HPA** sees CPU > target → scales to 8 pods → new pods pass readiness → join the Service
5. A node dies → controllers notice → pods rescheduled elsewhere → users notice nothing
6. You deploy v2 → rolling update swaps pods gradually → old pods get SIGTERM, finish in-flight requests (**graceful shutdown**), then exit

Nobody touched a server. That's the point.

---

## 6. Kubernetes' Role in Spring Boot Cloud Native Development

**Spring Boot makes the app; Kubernetes runs the fleet.**

Spring Boot is already a good K8s citizen — your job is wiring the two together correctly:

| Kubernetes needs | Spring Boot provides |
|---|---|
| Self-contained app | Embedded server — the container IS the runtime |
| Health probes | Actuator: `/actuator/health/liveness` + `/readiness` |
| Externalized config | `application.yml`, env vars, Spring profiles ← ConfigMaps/Secrets |
| Metrics for autoscaling | Micrometer → `/actuator/prometheus` |
| Graceful shutdown | `server.shutdown=graceful` (finish in-flight requests on SIGTERM) |
| No local state | State goes to Postgres/Redis/Kafka — pods stay disposable |

**The division of labor in a cloud native Spring Boot system:**

- **Spring Boot** = application logic, APIs, business rules, resilience patterns (Resilience4j)
- **Kubernetes** = placement, healing, scaling, networking, rolling deploys
- **Spring Cloud** = the bridge where needed (config, gateway, discovery)

**Java-specific K8s gotchas (interview gold):**

- **Startup probes** — JVM boot time kills apps under naive liveness probes
- **CPU limits** — throttling during JIT warmup makes Java apps slow exactly when scaling; many teams set requests but no CPU limit
- **`-Xmx` ≠ container memory** — JVM uses heap + metaspace + threads + off-heap; mis-sizing is the #1 cause of Java OOMKills. Use `MaxRAMPercentage` on modern JVMs
- **Image size** — slim JRE bases, or GraalVM native images for instant startup (matters for autoscaling/serverless)

### Does Kubernetes use the `/actuator/shutdown` endpoint?

**No — never expose it in a cluster.** Kubernetes doesn't stop your app via HTTP; it sends **SIGTERM** to PID 1 (the JVM). Spring Boot registers JVM shutdown hooks automatically, so SIGTERM alone triggers an orderly shutdown — the endpoint is a pre-orchestration relic, and exposing it is a security smell (anyone who can reach it can kill your app).

What you actually need for graceful pod termination:

```properties
server.shutdown=graceful                            # stop accepting new req, drain in-flight
spring.lifecycle.timeout-per-shutdown-phase=20s     # max drain wait
```

```yaml
terminationGracePeriodSeconds: 40   # must EXCEED app drain time, else SIGKILL mid-request
```

**The three termination gotchas (interview gold):**

1. **Endpoint-removal race** — SIGTERM and Service-endpoint removal happen ~simultaneously; requests can still hit a dying pod. Fix: `preStop` hook (`sleep 5`) delays SIGTERM until endpoint propagation completes.
2. **Dockerfile exec form** — `ENTRYPOINT ["java","-jar","app.jar"]` makes java PID 1 so it *receives* SIGTERM. Shell form (`ENTRYPOINT java -jar app.jar`) makes bash PID 1 — signal dies there, app gets SIGKILLed later.
3. **Grace period mismatch** — if your drain time exceeds `terminationGracePeriodSeconds` (default 30s), in-flight requests die anyway.

---

## 7. What Breaks Looks Like (Recognize These States)

| Status | Meaning | First thing to check |
|---|---|---|
| `CrashLoopBackOff` | Container starts, dies, repeats | `kubectl logs --previous` + events |
| `OOMKilled` | Exceeded memory limit | JVM sizing vs limits |
| `ImagePullBackOff` | Can't fetch image | Image name/tag, registry auth |
| `Pending` | Scheduler can't place it | `describe pod` → events (usually resources) |
| Pod running but no traffic | Readiness failing / Service selector mismatch | probe config, labels |
| `Evicted` | Node ran out of resources | resource requests, QoS class |

**The master debugging habit**: `kubectl describe pod` → read the **Events** section. It answers ~80% of "why is this broken."

---

## 8. What You Do NOT Need (Scope Check for Backend Devs)

You're learning to *run workloads on* Kubernetes, not to *administer* Kubernetes.

| Defer / skip | Focus instead |
|---|---|
| etcd internals, control plane ops | Pods, Deployments, Services, probes |
| CNI/networking plumbing | Resources, autoscaling, debugging |
| Cluster provisioning/upgrades | ConfigMaps, Secrets, Ingress |
| Deep RBAC/admission controllers | Zero-downtime deploys, Helm basics |

Cert fit: **CKAD** (app developer) matches this scope; **CKA** is the admin track.

---

## 9. The Big Three Deep-Dive: Pods, Deployments, Services — for Spring Boot Developers

You write `app.jar`. These three objects exist to answer one question: **how do we run many copies of your jar safely?**

**The layman frame:**

- **Pod** = **one running instance of your app.** `java -jar payment-api.jar` on your laptop is essentially one pod — JVM + embedded Tomcat wrapped up. *One employee on shift.*
- **Deployment** = **the shift manager.** "Always keep 3 copies of payment-api running. When v2.0 arrives, swap them in gradually. If one dies, hire a replacement instantly." You never create pods directly — the Deployment creates and supervises them.
- **Service** = **the company's phone number.** Pods die and get new IPs constantly, so hardcoding `http://10.0.1.5:8080` in your `WebClient` is useless. A Service gives a stable name — `http://payment-service` — that always reaches a healthy pod, load-balanced. It replaces what you'd otherwise need Eureka/discovery + a load balancer for.

The glue between them is **labels**: the Deployment tags its pods `app=payment-api`; the Service routes to anything carrying that tag.

### Pod — "one copy of your app, running"

**What it really is**: when you run `java -jar payment-api.jar` on your laptop, that JVM process is essentially what a pod is — your Spring Boot app (JVM + embedded Tomcat + jar) running inside a container, wrapped with an IP address and lifecycle rules.

**Layman analogy**: one employee on shift. Trained (your code), at a desk (its node), reachable at an extension (its pod IP).

**What a Spring Boot dev must internalize:**

- The container inside the pod = your app image. `server.port=8080` → `containerPort: 8080`.
- **Its IP is disposable.** The pod dies, a replacement comes up with a *different* IP. Anything in your code that assumes a stable address — hardcoded URLs, in-memory session state, local file writes — is a bug waiting to happen.
- **In-memory state is a trap.** `HashMap` caches, `@SessionScope`, files in the working directory — all vanish on restart AND are invisible to sibling replicas. State belongs in Postgres/Redis. This is why "stateless" keeps coming up.
- **You almost never write a Pod YAML by hand** — a bare pod that dies stays dead. No supervisor. That's the Deployment's job.

### Deployment — "the shift manager for your app"

**What it really is**: a supervisor loop that owns your pods — how many copies, which image version, and how updates roll out.

**Layman analogy**: a shift manager who keeps exactly 3 cooks in the kitchen, instantly replaces anyone who walks out, and when the new recipe arrives, swaps cooks in gradually so the kitchen never stops serving.

**What it manages for your Spring Boot app:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 3                        # "always keep 3 JVMs running"
  selector:
    matchLabels:
      app: payment-api               # manages pods carrying this tag
  template:
    metadata:
      labels:
        app: payment-api             # ← THE GLUE: tags every pod it creates
    spec:
      containers:
      - name: payment-api
        image: payment-api:1.0       # your dockerized jar
        ports:
        - containerPort: 8080        # == server.port=8080
        envFrom:
        - configMapRef:
            name: payment-config     # your externalized application.yml
        readinessProbe:              # "can you take traffic?"
          httpGet:
            path: /actuator/health/readiness
            port: 8080
        livenessProbe:               # "are you alive or stuck?"
          httpGet:
            path: /actuator/health/liveness
            port: 8080
        startupProbe:                # "have you finished JVM boot?"
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          failureThreshold: 30       # generous — Java starts slowly
        resources:
          requests:
            memory: "512Mi"
          limits:
            memory: "1Gi"            # JVM dies (OOMKilled) past this
```

**Read it as**: this file is the *operational* half of your `application.yml`. Replicas, version, env vars, health checks, memory — everything ops-side, declared.

**The killer features:**

- **Rolling update**: `image: payment-api:2.0` → K8s swaps pods gradually — new pod comes up, passes readiness, *then* an old pod is removed. Users see zero downtime. This replaces the "stop server, replace war, start server, pray" era.
- **Rollback**: bad release → `kubectl rollout undo deployment/payment-api` → previous version restored.
- **Self-healing**: pod dies → the reconciliation loop notices "2 running, wanted 3" → replacement scheduled in seconds.

### Service — "the company's phone number"

**What it really is**: pods die and get new IPs constantly, so `http://10.0.1.5:8080` is useless tomorrow. A Service is a **stable virtual IP + DNS name** that load-balances across whichever matching pods are currently alive.

**Layman analogy**: calling the restaurant's phone number. You don't call a specific cook's cell phone — you call the front desk, and whoever's on shift picks up. Cooks come and go; the number never changes.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service          # ← becomes a DNS name inside the cluster
spec:
  selector:
    app: payment-api             # routes to pods with this label
  ports:
  - port: 80
    targetPort: 8080             # your server.port
```

**What it means for your Spring Boot code:**

```java
// BEFORE (hardcoded — breaks constantly)
WebClient.create("http://10.0.1.5:8080");

// AFTER (Service DNS — always works)
WebClient.create("http://payment-service");
// K8s DNS resolves 'payment-service' → Service IP → load-balances to a healthy pod
```

- **It replaces service discovery + a load balancer in one.** No Eureka, no Ribbon, no external LB between internal services — `http://order-service`, `http://inventory-service` just work via cluster DNS.
- **Readiness is what controls it.** A pod failing its readiness probe is automatically pulled from the Service's endpoints — traffic stops reaching it, *without restarting it*. That's how rolling updates drop zero requests.
- One Service fronts *all* replicas — clients never know or care whether 3 or 30 pods are behind it.

### How the three connect — the label is the glue

```
Deployment (app=payment-api) ──creates/supervises──> Pod 1, Pod 2, Pod 3
                                                        │  all tagged app=payment-api
Service (selector: app=payment-api) ──load-balances───> ┘ whichever pods are alive + ready
```

| Object | Your world equivalent | One-liner |
|---|---|---|
| Pod | `java -jar app.jar` | One running copy — disposable |
| Deployment | systemd/PM2 + release manager | Keeps N copies alive, rolls versions safely |
| Service | Eureka + load balancer, built-in | One stable name → many ephemeral pods |

**The workflow as a dev**: you build `payment-api:2.0` (dockerized jar) → update the Deployment's image → K8s handles everything else: new pods boot, probes gate them, Service shifts traffic, old pods drain gracefully. Your job is the jar and the YAML; their job is the fleet.

---

## 10. Glossary

| Term | Plain meaning |
|---|---|
| Cluster | The whole fleet — control plane + worker nodes |
| Pod | Smallest deployable unit; wrapper around your container |
| Reconciliation loop | The endless "compare desired vs actual, fix differences" cycle |
| Desired state | What you declared in YAML; K8s' job to maintain |
| Kubelet | The node's local manager taking orders from the control plane |
| Endpoints | The live list of pods behind a Service |
| Rolling update | Gradual pod-by-pod replacement → zero-downtime deploy |
| OOMKilled | Pod killed for exceeding its memory limit |
| Helm | Package manager for K8s — templates YAML into reusable "charts" |
| kind / minikube | Run a real K8s cluster locally for learning |
