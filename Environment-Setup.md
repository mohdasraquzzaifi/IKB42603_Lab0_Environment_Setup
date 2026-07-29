# Lab 0: Environment Setup Report

## Course Information

| Item | Details |
| --- | --- |
| Course | IKB42603 Cloud Computing Security Essentials |
| Lab | Lab 0 - Environment Setup |
| Student Name | MUHAMMAD ASRA QUZZAIFI BIN MOHD RABI |
| Date | 29 July 2026 |

## Objective

To prepare and verify a local cloud-computing security environment using
Docker, AWS CLI v2, kind, kubectl, helper tools, LocalStack, and Kubernetes.

## Learning Outcomes

Upon completing this lab, the student is able to:

1. Install and verify the core tools used for local cloud and container work.
2. Create and inspect a local Kubernetes cluster using kind and kubectl.
3. Start LocalStack and verify its AWS-compatible services locally.
4. Configure AWS CLI v2 to communicate with LocalStack using test credentials.

## Environment

The environment used was a Kali Linux terminal. The completed setup includes:

- Docker version `28.5.2`
- AWS CLI version `2.36.9`
- kind version `0.23.0`
- kubectl client version `v1.33.4`
- Kustomize version `v5.5.0`
- OpenSSL `3.5.5`, oathtool `2.6.12`, and copyparty `1.28.2`
- LocalStack version `3.0.2`
- A kind Kubernetes cluster named `ccse`

## Step-by-Step Implementation

### 1. Install and Verify Docker

Docker provides the container runtime needed by LocalStack and kind. After
installation, verify Docker with:

```bash
docker --version
```

**Result:** Docker version `28.5.2` was installed successfully.

**Evidence:** <img width="612" height="62" alt="DOCKER" src="https://github.com/user-attachments/assets/0815ac18-5df9-418e-8c52-58df63be0f84" />


### 2. Install and Verify AWS CLI v2

AWS CLI v2 is used to issue AWS commands against LocalStack. Verify the
installation with:

```bash
aws --version
```

**Result:** AWS CLI version `2.36.9` was installed successfully.

**Evidence:** <img width="610" height="61" alt="AWS" src="https://github.com/user-attachments/assets/eab97bf1-b30c-4c85-accf-38df605b59a3" />


### 3. Install and Verify kind and kubectl

kind creates a local Kubernetes cluster using Docker, while kubectl manages and
queries that cluster. Verify both tools with:

```bash
kind --version
kubectl version --client
```

**Result:** kind `0.23.0`, kubectl client `v1.33.4`, and Kustomize `v5.5.0`
were available.

**Evidence:** <img width="216" height="100" alt="KIND KUBECTL" src="https://github.com/user-attachments/assets/fb23860a-bb21-41f2-8b9f-5d644b1c067c" />


### 4. Install and Verify Helper Tools

The helper tools support security and lab activities. Verify them with:

```bash
openssl version
oathtool --version
copyparty --version
```

**Result:** OpenSSL `3.5.5`, oathtool `2.6.12`, and copyparty `1.28.2` were
available.

**Evidence:** <img width="622" height="197" alt="HELPER TOOLS" src="https://github.com/user-attachments/assets/d61cb72e-6b71-4dde-9b0c-ece0b80b5760" />


### 5. Start and Verify LocalStack

Start LocalStack according to the lab cheatsheet. Then check that its local
AWS-compatible services are healthy:

```bash
curl http://localhost:4566/_localstack/health
```

**Result:** LocalStack version `3.0.2` responded successfully, and the listed
services were reported as `available`.

**Evidence:** <img width="622" height="292" alt="START AND VERIFY LOCALSTACK" src="https://github.com/user-attachments/assets/db8fcd96-5b68-409c-834b-723a921f52a9" />


### 6. Create and Verify the Kubernetes Cluster

Create the kind cluster named `ccse` as specified in the cheatsheet. Verify the
cluster and its node with:

```bash
kubectl cluster-info --context kind-ccse
kubectl get nodes
```

**Result:** The Kubernetes control plane and CoreDNS were running. The
`ccse-control-plane` node was in `Ready` status with Kubernetes version
`v1.30.0`.

**Evidence:** <img width="845" height="190" alt="CREATE AND VERIFY THE KUBERNETES CLUSTER" src="https://github.com/user-attachments/assets/e3649b3e-c07c-4af2-a641-3a38d4a2e172" />


### 7. Configure AWS CLI to Use LocalStack

Configure AWS CLI with LocalStack test credentials and the required region:

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```

Set the LocalStack endpoint and test the AWS CLI connection:

```bash
EP="--endpoint-url=http://localhost:4566"
aws sts get-caller-identity --endpoint-url http://127.0.0.1:4566
```

**Result:** The command returned the LocalStack test AWS identity, including
account ID `000000000000`, confirming that AWS CLI was configured correctly.

**Evidence:** <img width="567" height="262" alt="CONFIGURE AWS CLI TO USE LOCALSTACK" src="https://github.com/user-attachments/assets/e2b5c3bb-baee-4fc9-a411-a013203c5abf" />


## Conclusion

The Lab 0 environment was set up and verified successfully. Docker, AWS CLI
v2, kind, kubectl, helper tools, LocalStack, and the `ccse` Kubernetes cluster
are ready for subsequent IKB42603 Cloud Computing Security Essentials labs.
