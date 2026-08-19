# IKB42603 Lab 2: Secure Isolation and Multi-Tenancy

## Purpose

This report documents the environment setup and verification activities for **IKB42603 Lab 2 – Secure Isolation and Multi-Tenancy**. The lab demonstrates isolation across three dimensions:

- **Compute:** separate tenants through Kubernetes namespaces and limit shared resources.
- **Network:** show the default-open risk, then block cross-tenant ingress using a default-deny policy.
- **Storage:** restrict access to tenant secrets with RBAC and discuss secure deletion.

## Environment

The environment used Kubernetes in Docker (`kind`) with `kubectl`, Docker, and the Calico CNI. Calico was selected because Kubernetes NetworkPolicy objects require a network plugin that enforces them; the default kind network alone does not enforce these policies.

The cluster name was `ccse-lab2`, and the tenant namespaces were `tenant-a` and `tenant-b`.

## Step 1 – Create a kind cluster with policy enforcement

The default CNI was disabled while creating the kind cluster so Calico could be installed as the policy-enforcing CNI.

```sh
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s
```

**Result:** The cluster was created successfully and the Kubernetes context was set to `kind-ccse-lab2`. The Calico manifest was applied; its resources, including the Calico daemonset and controllers, were created.

![Figure 1. kind cluster ccse-lab2 created.](<cluster with policy enforcement.png>)

![Figure 2. Calico policy-enforcement components applied.](<cluster with policy enforcement 2.png>)

## Step 2 – Create two isolated tenant namespaces

Two namespaces model two customers sharing one physical cluster.

```sh
kubectl create namespace tenant-a
kubectl create namespace tenant-b
```

**Result:** Both `tenant-a` and `tenant-b` were created.

![Figure 3. Creation of tenant-a and tenant-b namespaces.](<create 2 tenants.png>)

## Step 3 – Deploy a web service for each tenant

An NGINX deployment and ClusterIP service were created in each namespace.

```sh
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx
kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80
kubectl get pods,svc -n tenant-a
```

**Result:** The evidence shows the deployment and service were created in both namespaces. The `tenant-a` service is a ClusterIP service on port 80.

![Figure 4. Web deployments and services created for both tenants.](<deploy web server.png>)

## Step 4 – Demonstrate the default-open network risk

Before a NetworkPolicy was applied, the ClusterIP of `tenant-b`'s web service was retrieved and tested from `tenant-a`.

```sh
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo
# Returned: 10.96.152.159

kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://10.96.152.159 -o /dev/null -w 'HTTP %{http_code}\n'
```

**Result:** The probe returned `HTTP 200`. This proves that a workload in `tenant-a` could reach the service in `tenant-b` by default. Namespace separation alone therefore does not provide network isolation in a multi-tenant cluster.

![Figure 5. Cross-tenant probe succeeds before the policy (HTTP 200).](<curl tenant.png>)

## Step 5 – Apply a resource quota to tenant-a

The following ResourceQuota prevents `tenant-a` from consuming more than five pods, one requested CPU, or 512 MiB of requested memory.

```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF

kubectl describe resourcequota tenant-a-quota -n tenant-a
```

**Result:** `tenant-a-quota` was created. The description confirms the configured hard limits: five pods, one CPU request, and 512 MiB memory request. This control helps contain a noisy-neighbour tenant on shared compute infrastructure.

![Figure 6. Resource quota created and verified for tenant-a.](<resources quotas.png>)

## Step 6 – Apply default-deny ingress to tenant-b

The following NetworkPolicy selects every pod in `tenant-b` and denies all inbound traffic unless a separate allow policy is added.

```sh
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
```

**Result:** The `default-deny-ingress` policy was created in `tenant-b`.

![Figure 7. Default-deny ingress NetworkPolicy applied to tenant-b.](<session b task 4.png>)

## Step 7 – Verify that cross-tenant traffic is blocked

The same target service (`10.96.152.159`) was tested after applying the policy.

```sh
kubectl -n tenant-a exec -it deploy/web -- \
  curl -s -m 5 http://10.96.152.159 -o /dev/null -w 'HTTP %{http_code}\n'
```

**Result:** The command ended with `HTTP 000` and curl exit code `28`, which indicates a timeout. Compared with the earlier `HTTP 200`, this is direct before/after evidence that the default-deny policy blocked traffic from `tenant-a` to `tenant-b`.

![Figure 8. Cross-tenant request times out after the policy (HTTP 000; exit code 28).](<session b task 4 (2).png>)

## Step 8 – Enforce per-tenant secret access with RBAC

Each tenant received a secret. A service account named `app-a` was then granted a `reader` Role only in `tenant-a`.

```sh
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B

kubectl -n tenant-a create serviceaccount app-a
kubectl -n tenant-a create role reader --verb=get --resource=secrets
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a

SA=system:serviceaccount:tenant-a:app-a
kubectl auth can-i get secrets -n tenant-a --as=$SA
kubectl auth can-i get secrets -n tenant-b --as=$SA
```

**Result:** The authorization checks returned `yes` in `tenant-a` and `no` in `tenant-b`. This confirms that `app-a` can read secrets only in its own tenant namespace.

![Figure 9. RBAC permits tenant-a secrets and rejects tenant-b secrets.](<session b task 5.png>)

## Step 9 – Demonstrate data remanence handling

A Docker volume was used to create and normally delete sample sensitive data, then to overwrite data before deletion.

```sh
# Normal deletion and scan
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
   grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'

# Overwrite before deletion
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
   dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; echo wiped'
```

**Result:** The first command completed and printed `scan-done`; no recoverable plaintext was printed by this directory scan. The second command overwrote 1024 bytes with zeroes, removed the file, and printed `wiped`.

Normal deletion removes a directory entry but may not immediately erase underlying storage blocks. In cloud environments, users normally cannot control those physical blocks. Therefore, **cryptographic erasure**—destroying the encryption key—is the preferred practical method for making encrypted cloud data unrecoverable.

![Figure 10. Remanence scan completed and the sample file was overwritten before deletion.](<session b task 6.png>)

## Verification summary

| Control | Evidence and outcome |
| --- | --- |
| Compute separation | `tenant-a` and `tenant-b` were created as separate namespaces. |
| Default-open risk | Cross-tenant web request returned `HTTP 200` before a policy. |
| Resource isolation | `tenant-a-quota` limits pods, CPU requests, and memory requests. |
| Network isolation | After default-deny ingress, the same request returned `HTTP 000` and timed out. |
| Storage/secret isolation | `app-a` was authorized in `tenant-a` (`yes`) and denied in `tenant-b` (`no`). |
| Data remanence | Secure-wipe command overwrote 1 KiB before deletion; cryptographic erasure remains the preferred cloud approach. |

## Short answers

### Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous?

Kubernetes namespaces are primarily logical administrative boundaries; they do not automatically impose network firewall rules. Without a NetworkPolicy, pods can generally communicate across namespaces through routable Pod or Service addresses. In a multi-tenant cluster, this can expose services to another tenant and increases the risk of lateral movement after a compromise.

### Q2. Explain the default-deny principle and how the policy implements it.

Default-deny means traffic is blocked unless an explicit rule permits it. `default-deny-ingress` selects all pods in `tenant-b` and declares ingress policy without any allow rules. Calico therefore blocks inbound traffic, including the probe from `tenant-a`, until a deliberately scoped allow policy is added.

### Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?

Containers share the host kernel, whereas virtual machines use a hypervisor boundary and separate guest kernels. A VM boundary is appropriate for untrusted or high-risk tenants, strong compliance separation, different operating-system requirements, or workloads where kernel-level compromise has an unacceptable impact.

### Q4. What is data remanence, and why is cryptographic erasure preferred in the cloud?

Data remanence is residual data that can remain on storage media after normal deletion. Cloud customers usually cannot physically overwrite every underlying block because storage is abstracted and may be replicated. Destroying the key for strongly encrypted data is faster, scalable, and makes the ciphertext unusable.

### Q5. Which isolation dimensions did each task exercise?

| Task | Isolation dimension |
| --- | --- |
| Cluster setup and tenant namespaces | Compute / logical tenant separation |
| Default-open probe | Network (identifies the missing control) |
| ResourceQuota | Compute / resource isolation |
| Default-deny NetworkPolicy and re-test | Network isolation |
| Per-tenant secrets with RBAC | Storage and access isolation |
| Normal deletion and overwrite | Storage / data remanence |

## Cleanup (only after assessment evidence has been retained)

```sh
kind delete cluster --name ccse-lab2
docker volume rm ccse-vol
```
