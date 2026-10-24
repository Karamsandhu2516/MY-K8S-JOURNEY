\## Day 8: ConfigMaps and Secrets



Welcome to Day 08 of my K8S journey! Today, we'll dive into ConfigMaps and Secrets.



\- \*\*ConfigMaps\*\* — a way to manage external configuration separately from container images.

\- \*\*Secrets\*\* — Kubernetes objects designed to store and manage sensitive data.



\---



\## What is a ConfigMap?



A ConfigMap is a Kubernetes object that stores key-value pairs of configuration data. It decouples configuration from application code, enabling updates without redeploying containers.



Diagram:



```text

\&#x20;   ConfigMap

\&#x20;       |

\&#x20;       v

\&#x20;      Pod

(Environment Variables / Volume Mounts)

```



\---



\## Why Do We Need ConfigMaps?



\- \*\*Separation of concerns\*\* → Keep configuration outside container images

\- \*\*Flexibility\*\* → Update configs without rebuilding images

\- \*\*Reusability\*\* → Share configs across multiple Pods

\- \*\*Portability\*\* → Run the same image with different configs in dev, test, prod



\---



\## Common ConfigMap Issues



\- \*\*ConfigMap not found\*\* → Pod fails if ConfigMap doesn't exist

\- \*\*Key mismatch\*\* → Environment variable not injected correctly

\- \*\*Volume mount path conflict\*\* → Overwrites existing files in container (the mounted volume hides the existing content of that directory; use `subPath` to mount a single file)



\---



\## Interview Questions



1\. \*\*What is a ConfigMap in Kubernetes?\*\*

&#x20;  → A key-value store for external configuration.

2\. \*\*How do ConfigMaps differ from Secrets?\*\*

&#x20;  → ConfigMaps store non-sensitive data; Secrets store sensitive data.

3\. \*\*How can ConfigMaps be consumed by Pods?\*\*

&#x20;  → As environment variables or mounted volumes.

4\. \*\*What happens if a ConfigMap is updated?\*\*

&#x20;  → Environment variables do not change until the Pod is restarted. Volume-mounted ConfigMaps are updated automatically after a short delay (not when mounted with `subPath`), but the application must reload the file to use the new values.

5\. \*\*Are ConfigMaps Namespace-scoped?\*\*

&#x20;  → Yes, they exist only within a single Namespace.



\---



\## Key Takeaways (ConfigMaps)



\- ConfigMaps store external configuration as key-value pairs

\- They decouple configuration from container images

\- Pods can consume ConfigMaps via environment variables or volumes

\- ConfigMaps are Namespace-scoped and non-sensitive

\- They are essential for flexible, portable workloads



\---



\## What is a Secret?



A Secret is a Kubernetes object that stores sensitive information in base64-encoded format. It prevents exposing credentials directly in Pod specs or container images.



> \\\*\\\*Important:\\\*\\\* base64 is encoding, not encryption. Anyone who can read a Secret can decode it.



Diagram:



```text

Secret

\&#x20;  |

\&#x20;  v

Pod (Environment Variables / Volume Mounts)

```



\---



\## Why Do We Need Secrets?



\- \*\*Security\*\* → Protect sensitive data like passwords and tokens

\- \*\*Separation of concerns\*\* → Keep secrets outside container images

\- \*\*Flexibility\*\* → Update secrets without rebuilding workloads

\- \*\*RBAC control\*\* → Restrict access to secrets per Namespace



\---



\## Common Secret Issues



\- \*\*Invalid base64 in the `data` field\*\* → Secret creation fails (use `stringData` or `kubectl create secret` to avoid this)

\- \*\*Wrong key reference\*\* → Pod fails to inject secret

\- \*\*RBAC restrictions\*\* → User lacks permission to access secrets

\- \*\*Mount path conflicts\*\* → Overwrites existing files in container



\---



\## Key Takeaways (Secrets)



Today you learned:



\- Secrets store sensitive data like passwords and tokens

\- They decouple sensitive information from container images

\- Pods can consume Secrets via environment variables or volumes

\- Secrets are Namespace-scoped and must be protected with RBAC

\- They are essential for secure Kubernetes workloads



\---



\## Lab: ConfigMap and Secret in a Pod (Added)



\###  Create a ConfigMap



```bash

kubectl create configmap app-config --from-literal=APP\\\_ENV=dev --from-literal=APP\\\_COLOR=blue

kubectl get configmap app-config -o yaml

```



\###  Create a Secret



```bash

kubectl create secret generic db-secret --from-literal=DB\\\_USER=admin --from-literal=DB\\\_PASSWORD='S3cr3t!'

kubectl get secret db-secret -o yaml

```



The values in the output are base64-encoded. Anyone can decode them:



```bash

echo 'YWRtaW4=' | base64 --decode

```



```text

admin

```





```bash

kubectl apply -f config-demo.yaml

kubectl exec config-demo -- env | grep -E "APP\\\_|DB\\\_"

kubectl exec config-demo -- ls /etc/db-secret

```



\- `envFrom` injects every key of the ConfigMap as an environment variable.

\- `secretKeyRef` injects one key from the Secret as an environment variable.

\- The `secret` volume creates one file per key under `/etc/db-secret`.



\---



\## How ConfigMaps and Secrets Work Under the Hood (Added)



\- \*\*Stored in etcd:\*\* both are normal API objects. The API server saves them in etcd like any other object.

\- \*\*Secrets are not encrypted by default:\*\* they are only base64-encoded in etcd. To protect them you must enable \*\*encryption at rest\*\* (an `EncryptionConfiguration` on the API server, or a KMS provider). Managed clusters like EKS support envelope encryption with KMS.

\- \*\*Kubelet delivers them:\*\* when a Pod using a ConfigMap or Secret is scheduled to a node, the kubelet fetches the data from the API server.

\- \*\*Environment variables:\*\* injected when the container starts. They are fixed, so changes never reach a running container.

\- \*\*Volumes:\*\* the kubelet writes the data as files. Secret volumes use `tmpfs` (memory), so Secret data is not written to the node's disk. The kubelet refreshes mounted files periodically, so updates appear after a short delay (not with `subPath`).

\- \*\*Same Namespace only:\*\* a Pod can only reference a ConfigMap or Secret that is in its own Namespace.

\- \*\*Size limit:\*\* each object can hold up to 1 MiB of data.

\- \*\*What happens if it's missing:\*\*

&#x20; - Missing ConfigMap/Secret or key used as an environment variable → Pod shows `CreateContainerConfigError`

&#x20; - Missing ConfigMap/Secret used as a volume → Pod stays in `ContainerCreating` with a `FailedMount` event (check `kubectl describe pod`)

\- \*\*Secret types:\*\* `Opaque` (default), `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson` (image registry credentials), `kubernetes.io/service-account-token`.



\---



\## Production Best Practices (Added)



1\. \*\*Remember: base64 is not security.\*\* Anyone with `get` access to Secrets can read them, and so can anyone who can create a Pod in that Namespace (they can mount the Secret).

2\. \*\*Enable encryption at rest\*\* for etcd.

3\. \*\*Use least-privilege RBAC.\*\* Avoid giving `get`, `list` or `watch` on Secrets broadly.

4\. \*\*Never commit Secret YAML to Git.\*\* Use tools like Sealed Secrets, SOPS, External Secrets Operator, or an external store such as HashiCorp Vault or AWS Secrets Manager.

5\. \*\*Prefer volume mounts over environment variables for Secrets.\*\* Environment variables can leak through logs, crash dumps and child processes.

6\. \*\*Roll out changes safely.\*\* After changing a ConfigMap or Secret used as environment variables, restart the workload: `kubectl rollout restart deployment <name>`.

7\. \*\*Use immutable ConfigMaps and Secrets\*\* (`immutable: true`) for values that should never change. This prevents accidental edits and reduces load on the API server.

8\. \*\*Keep one set per environment\*\*, in separate Namespaces for dev, test and prod.

9\. \*\*Enable audit logging\*\* to track who accessed Secrets.



\---



\## Interview Questions: Secrets (Added)



1\. \*\*Are Secrets encrypted in Kubernetes?\*\*

&#x20;  → Not by default. They are base64-encoded in etcd. Encryption at rest must be enabled separately.

2\. \*\*Environment variables or volumes for Secrets?\*\*

&#x20;  → Volumes are generally safer, and mounted files can be refreshed without restarting the Pod. Environment variables can leak and need a restart to change.

3\. \*\*A Pod is stuck in `CreateContainerConfigError`. What do you check?\*\*

&#x20;  → Whether the referenced ConfigMap, Secret or key exists in the same Namespace. Use `kubectl describe pod` to read the event.

