# Kubernetes Jobs: What They Are and How They Work

## 1. What is a Job in Kubernetes?

In Kubernetes (K8s), a **Job runs a task that needs to finish successfully**.

Unlike a Deployment, which keeps an application running, a Job runs a task until it completes.

### Example

Imagine you have an e-commerce application running in Kubernetes:

- **Frontend** — shows products.
- **Backend** — processes orders.
- **Database** — stores customer and order information.

Now imagine you need to back up data or generate a monthly sales report. This task does not need to run forever. It just needs to run, finish its work, and exit.

That's where a Kubernetes Job comes in.

```text
You create a Job
      |
      v
Kubernetes creates a Pod
      |
      v
The container runs the task
      |
      +---- Success ---> Job is marked Complete
      |
      +---- Failure ---> Kubernetes can retry the task
```

**Important:** A Job does not normally keep a Pod running forever. It makes sure the required task completes successfully, subject to its retry and failure settings.

## 2. Job vs Deployment vs CronJob

This is the most important distinction to understand.

| Resource | Purpose | Example |
|---|---|---|
| **Deployment** | Keeps an application running | Frontend, backend API, web server |
| **Job** | Runs a task to completion | One-time data processing, report generation, database migration |
| **CronJob** | Creates Jobs on a schedule | Nightly backup at 2 AM |

Easy way to remember:

- **Deployment** = keep the application running.
- **Job** = complete a task.
- **CronJob** = run a task on a schedule.

## 3. How does a Job work under the hood?

Suppose you run:

```bash
kubectl apply -f report-job.yaml
```

Here is what happens:

1. **API server:** Receives and validates your Job configuration.
2. **etcd:** Stores the desired Job state.
3. **Job controller:** Notices the new Job and creates a Pod to perform the task.
4. **Scheduler:** Assigns the Pod to a suitable node.
5. **Kubelet and container runtime:** Start the container on that node.
6. **Your script runs:** It generates the report and exits with status code `0` if successful.
7. **Job controller:** Observes the completed Pod and marks the Job as `Complete`.

If the container exits with an error, Kubernetes can retry it according to the Job's settings. Retries are not unlimited by default; Jobs have failure-handling limits.




### What do these fields mean?

| Field | Meaning |
|---|---|
| `kind: Job` | Create a Job, not a Deployment |
| `backoffLimit: 3` | Allow up to three counted Pod failures before the Job is considered failed |
| `restartPolicy: Never` | Do not restart the container inside the same Pod; the Job controller can create a replacement Pod |
| `image` | Container image that runs the task |
| `command` | The task to execute |

### Run and inspect the Job

```bash
kubectl apply -f report-job.yaml
kubectl get jobs
kubectl get pods
kubectl logs job/report-job
kubectl describe job report-job
```

Expected logs:

```text
Generating sales report
Report completed
```

After about 10 seconds, the Job should show `1/1` completions, assuming the image can be pulled and the Pod runs successfully.




| Task | Recommended Kubernetes resource |
|---|---|
| Run a web frontend continuously | Deployment |
| Run a backend API continuously | Deployment |
| Process a one-time batch of uploaded files | Job |
| Run a database schema migration during a release | Often Job |
| Back up a database every night | CronJob |
| Run a log collector on every node | DaemonSet |
| Run a stateful database with stable Pod identities | Often StatefulSet |

A database migration can be a Job, but you do not automatically create one for every deployment. Your release pipeline or deployment workflow must trigger it at the correct time.

**Note:** A Job completing does not mean its Pod and logs disappear immediately. Completed Pods normally remain until cleaned up or removed through a Job's TTL setting.

## Why is READY 0/1 if the Job succeeded?

Because the container has finished running. It is no longer running and ready to serve requests, but its completion status is recorded. This is expected for a Job.


## Key takeaway

Kubernetes controllers manage different workload lifecycles:

- **Deployment:** maintains long-running applications.
- **Job:** runs a finite task until completion.
- **CronJob:** schedules Jobs to run repeatedly.
- **DaemonSet:** runs a Pod on each eligible node.
- **StatefulSet:** manages workloads that need stable identities or storage.

