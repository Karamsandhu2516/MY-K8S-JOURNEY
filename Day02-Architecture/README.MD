## Day 2: Kubernetes Architecture

###  What is K8S and why required K8S?

**A:** Docker let you package the app into a container with its dependencies and version, libraries. But running that in production raises questions: what if the container crashes? what if traffic spikes and you need more copies? what if a server dies? K8S answers all of them automatically.

1. **Orchestration:** Schedule container in Pods across cluster of machines that are nodes deciding where each one runs based on available resources.
2. **Auto-healing:** If a container or Pod crashed, K8S detects it via liveness probes and restarts it automatically. If the whole node dies, K8S reschedule its Pods onto healthy nodes. Readiness probes stop traffic from hitting a pod that isn't ready yet.
3. **Scaling:** Horizontal Pod autoscaler adds/removes Pod replicas based on CPU/memory/custom metrics.
4. **Service discovery & load balancing:** Pod gets ephemeral IPs: a Service gives them a stable DNS name/IP and load balances traffic across replicas.
5. **Rolling updates and rollbacks:** deploy new version gradually with zero downtime, rollback instantly if something breaks.
6. **Config & Secret management:** ConfigMaps and Secrets decouple config/credentials from container images.
7. **Declarative state:** You describe the desired state in YAML, e.g. I want 3 replicas of this app running and the control plane, specially controller manager, constantly works to match reality of that desired state.

---

### Control Plane

Control plane is the brain of the cluster. It makes the global decisions like scheduling, detecting & responding to events.

#### Five components of Control Plane

#### 1. Kube-API Server

Kube API Server is the central management entity of K8s control plane. This is the only component that talks directly to ETCD. API Server is hub and everyone else spokes.

**Core functions:**

- Receives & validate requests (create, update, delete) for all K8s objects such as Pod, Services and namespaces.
- Authenticate and authorizes requests using mechanisms like certificate & RBAC.
- Act as a gateway for cluster communication between components.
- Support extensibility through Custom Resource Definitions (CRDs), enabling users to add new API's.

**Request Flow:** Request comes as `kubectl apply -f <Pod>` as HTTP API calls.

- The API server authenticate & authorizes the request with either certificate or token.
- The request is validated, changes are persisted by writing to ETCD.
- Relevant controllers (controller manager or scheduler) are noticed to act upon changes.

If API server dies no new request can be processed - no kubectl commands, no scheduling of new Pods, no scaling decisions. But already running pod on worker node keep running fine because Kubelet manage local containers independently. Cluster become frozen, no react to new events like crash/node failure until the API server is back.

**Recovery:** If one API server becomes unavailable, LB would remove the unhealthy instance, direct the request to remaining API Servers. In self managed cluster (like it's not EKS), control plane size is designed on the base of availability and workload requirements. For production multiple API servers instances are typically deployed behind load balancer. Tools like Kubeadm can bootstrap those control plane instances.

#### 2. ETCD

(/etc comes from Linux traditional dir stores system config files, d stands for distributed)

It is single source of truth for the entire K8s cluster. ETCD is the highly available distributed key-value store. It stores all cluster state and configurations such Pods, services, deployments, secrets and more. If ETCD is down API can't write any new request so no new Pod can be scheduled, no updates can be processed.

ETCD works on Raft consensus algorithm. ETCD always works or runs in odd numbers (3, 5, 7) so it can maintain majority. With 3 nodes it can tolerate 1 failure but if 2 dies etcd becomes read only, no writes. ETCD need to be recovered from previously taken etcd snapshot backups (`etcdctl snapshot restore`) - this is why regular backups are mandatory. Rebuild fresh etcd cluster from that snapshot. If no snapshot backup exists there's no way to recover the actual notebook data. It's cluster admin / SRE's job, usually scripted/automated regular etcd snapshots.

> **Node:** Node could be physical or virtual machine that has Kubectl, CNI (network setup) and container runtime (containerd) installed and configured to join a specific cluster.

#### 3. Kube-Scheduler

Decides where Pod runs: It watch the API Server for Pod with no node assigned yet. For each pod it picks the best node and writes that decision back to the API Server which persists it to etcd.

**How it select the node is two stage process:**

**1. Filtering:** Eliminate nodes that can't run the Pod at all.

- (a) Does the node have enough free CPU/memory to satisfied the Pod's resources requests?
- (b) Does the Pod have a "nodeSelector" or nodeaffinity rule that excludes this node?
- (c) Are there "taints" on the node that Pod doesn't "tolerate"?
- (d) Does Pod request a specific port that's already in use on this node?
- (e) Are there any volume/zone constraints - e.g. an EBS volume only mounts in one AZ, so node outside that AZ are filtered out.

**2. Scoring:** Rank the feasible nodes 0-10 on multiple weighted criteria:

- (a) **LeastRequestedPriority:** Prefer nodes with more free capacity.
- (b) **BalancedResourceAllocation:** Prefer nodes where CPU and memory usage stay balanced relative to each other.
- (c) **InterPod Affinity:** Prefer/avoid nodes based on labels on Pod, rather than on node labels.
  - (i) **Affinity:** Score nodes higher if they already run Pods matching a target label.
  - (ii) **Anti-affinity:** Score nodes higher if they do not contain Pods matching a target label.

The Kube Scheduler doesn't launch the container. It just writes a Binding object on the API Server saying Pod X - Node Y, that's it, it's job is done. The Kubelet (worker node component) picks this up via it's own watch and actually pulls the image and starts it via the container runtime.

If it dies new Pod would stuck on Pending. Scheduled Pods would be unaffected. Nothing will be assigned to the node until the scheduler comes back.

**Recovery:** Since it's stateless and just watches/writes via API, simply restarting the process is sufficient. Like depend on cluster:

- (a) If it is running as static Pod, "Kubelet" detects the container process exit and automatically restarts it via the container runtime (containerd).
- (b) If scheduler runs directly as System service, Systemd detects process termination and restarts it based on its unit file configuration.

(High availability & Leader Election)
Also, in production cluster there are multiple replicas of Scheduler across control plane nodes (typically 3). However only one instance actively schedules Pods at given time.

#### 4. Kube-Controller Manager

It is a daemon running in control plane that embeds many core control loops (controllers). These controllers continuously monitor the state of the cluster such as nodes, Pods and other resources and work to maintain the desired state as specified in cluster's configuration. If the current cluster state deviates from the desired state the controller manager triggers actions to reconcile the difference. Each controller is a separate process but to reduce complexity they are all compiled into a single binary and run in a single process.

There are different types of controllers:

- (i) **Node controller:** Responsible for noticing and responding when node go down. Reschedule Pods somewhere else. (Autohealing)
- (ii) **Job controller:** watches the job objects like workloads designed to run a set of tasks. Creates the Pods to run those tasks to completion.
- (iii) **Endpoint / EndpointSlice controller:** Keep Service-to-Pod IP mappings up to date as Pods are ephemeral in nature.
- (iv) **ReplicaSet controller:** this is what powers Pod-level self-healing. It continuously checks: Deployment X wants 3 replicas, I only see 2 Pods → Create a new Pod → goes back to the scheduler → gets placed on a node.

If it dies self healing property is effectively paused. Existing healthy Pods keep running.

**Recovery:** Restarts or standby takes leadership.

#### 5. Cloud-Controller Manager

Bridge the K8s to cloud provider. Handles things like provisioning a cloud Load balancer when we create a Service type: LoadBalancer, attaching cloud disks to the correct node and syncing node metadata (region/zone/instance-type) from the cloud API.

If it dies cloud specific actions stall like provisioning new ELB, but the core scheduling/Pod management is unaffected.

---

Everything is API centric. There is no direct RPC (remote procedure call) except to etcd, which only API server touches. Scheduler, controller manager and Kubelets all just:

**watch API server → detect relevant change → act → write result back to API server.**

If entire control plane dies: worker nodes keep running existing Pods - Kubelet operates independently once it has a Pod spec and container runtime (containerd) keeps containers alive. The App stays up. But no new deployments, no new scaling, no self healing of crashed Pods/nodes.

**Recovery:**

- (i) If it is managed service, cloud provider manages control plane. You generally can't lose it entirely.
- (ii) **Self-managed:** Restore etcd from latest snapshot backup, redeploy the control plane components or IaC (Gitops pipeline) used to reconnect based on restored state.

> **Note:** If no etcd backup exists, total loss of cluster state. Only path to rebuild from version controlled manifests. Git repo would become the disaster recovery source of truth even if etcd is gone.

---

### Worker Node

If control plane decides what need to be made and where then worker node is where it actually gets cooked and served. Every worker node runs three key components:

#### 1. Kubelet

It is an agent that runs on every worker node. It's only component on worker node that talks to the Control plane. It watchs the apiserver constantly, when scheduler assign a Pod to a node the Kubelet notices, pulls the container image and tells the container runtime to start the container. It continuously checks is this container still alive and healthy via liveness/readiness probes. It keep update the status back to apiserver.

If a container crashed the Kubelet is one that notice first and restarts it locally, without even the control plane. The control plane only gets involved if the Kubelet itself or the whole node goes silent.

If Kubelet dies the node stop reporting it. The control plane's node controller waits for 40 sec (default) then mark node NotReady. If it stays down past a threshold, the controller manager reschedules them elsewhere.

#### 2. Container-Runtime

It is software that actually runs containers: pull the image, create the isolated process/filesystem, starts/stops it. Kubelets doesn't cook, it just tell the runtime what to cook.

#### 3. Kube-Proxy

Handle networking rules so that traffic sent to the Service gets routed to one of the actual Pods behind it - even though Pod's IPs constantly change as they get created/destroyed. It watches the API Server for Service/Endpoint changes and programs networking rules via IPtables on the node so packets get forwarded to the correct Pod IP, load balancing across replicas.

---

### CNI (Container Network Interface) and Calico (Pod-to-Pod networking)

CNI handles how Pod get an IP address and talk to other Pods at all even across different nodes.

K8S itself has no built-in networking implementation - it just calls out whatever CNI plugin you installed. So CNI is another plug socket for networking.

Calico is one popular CNI plugin (others Flannel, Cilium). It's responsible for:

- Assigning each Pod its own IP
- Making sure Pod on node A can talk to Pod on Node B
- Enforcing the network policies e.g. only the frontend pods can talk to the database Pods

Without CNI plugin installed, Pods literally cannot get IP addresses or talk to each other - it's required piece, not optional, even though it's third party add-on to core K8s.

---

### Namespace

A way to logically divide one cluster into isolated sections. Same cluster, same staff i.e. (control plane, nodes) but separate rooms - e.g. dev, prod.

Nothing runs in general, every Pod, deployment, service belongs to some namespace.

### ReplicaSet

This is actually object behind the self-healing behaviour we discussed - the controller manager runs the reconciliation loop for ReplicaSets specifically.

- You declare I want 3 replicas of this Pod template.
- The ReplicaSet controller (part of controller-manager) continuously counts: how many pods matching this template's label currently exists.

You never create the replicaset directly - you create deployment instead and it creates/manages the ReplicaSet for you.

### Deployment

A layer on top of ReplicaSet that adds versioned, controlled updates.

You update a deployment's Pod template (version 2). The deployment controller creates a new replicaset v2 and gradually scales it up while scaling old replicaset down. So zero DT. The old replicaset isn't deleted - it keeps around at 0 replicas which is exactly makes rollback instant.

**Deployment manages → ReplicaSet manages → Pods**

### Service

Pods are emphemeral, get new IP everytime they're recreated. A service gets one stable virtual IP and DNS name that never changes.

- It selects target Pod via the same label selector machanisum as replicasets - "route to any Pod labeled app = backend".
- Kube-proxy is what actually implement this - it watches for which Pods currently match that label, programs and networking rules so traffic to the Service IP gets load-balanced across whichever Pods are alive right now.

**Common Service types:**

- **ClusterIP:** only reachable inside the cluster
- **NodePort:** exposes it on a static port on every node's IP, for basic external access
- **LoadBalancer:** asks the cloud-controller-manager to provision an actual cloud load balancer

---

### Flow: `kubectl apply -f deployment.yaml` (Deployment, replicas: 3, image: v1)

- apiserver writes it to etcd
- Deployment controller creates the replicaset
- ReplicaSet controller notices 0 of 3 Pods exist → Create 3 Pod objects
- Kube scheduler assigns each to a node
- Kubelet on each node pulls the image via container runtime (CRI) and starts the container.
- CNI (Calico) gives each Pod an IP so they can talk to each other.

You create a Service with selector: `app = myapp` → Kube-proxy on every node programs rules so any traffic to the Service IP load balances across those 3 Pod IPs.