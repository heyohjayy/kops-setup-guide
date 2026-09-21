# Kubernetes Cluster on AWS with kOps

A practical, reproducible guide for provisioning a Kubernetes cluster on AWS using **kOps**, **kubectl**, **AWS CLI**, an **Amazon S3 state store**, and **Cilium CNI networking**.

This guide walks through the complete setup from preparing a dedicated Ubuntu administration server to configuring AWS access, installing the required tools, creating the kOps state store, generating SSH credentials, and preparing the environment for cluster creation.

The cluster built in this guide uses:

- **AWS** as the cloud provider
- **kOps** for Kubernetes cluster lifecycle management
- **Amazon S3** for persistent kOps cluster state
- **kubectl** for Kubernetes administration
- **Cilium** for cluster networking
- **Ubuntu** for the administration server and Kubernetes nodes
- **containerd** as the container runtime
- **1 control-plane node**
- **2 worker nodes**
- **Amazon EC2 `t3.medium` instances** for this lab

> [!IMPORTANT]
> Values such as cluster names, S3 bucket names, AWS regions, Availability Zones, instance types, IP addresses, Kubernetes versions, and EC2 instance IDs shown in this guide are examples from one deployment.
>
> Replace them with values appropriate for **your own AWS environment**.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [How kOps Builds the Cluster](#how-kops-builds-the-cluster)
3. [Prerequisites](#prerequisites)
4. [Step 1 — Prepare the kOps Administration Server](#step-1--prepare-the-kops-administration-server)
5. [Step 2 — Create a Dedicated kOps User](#step-2--create-a-dedicated-kops-user)
6. [Step 3 — Install and Configure AWS CLI](#step-3--install-and-configure-aws-cli)
7. [Step 4 — Install kOps](#step-4--install-kops)
8. [Step 5 — Install kubectl](#step-5--install-kubectl)
9. [Step 6 — Configure AWS Permissions](#step-6--configure-aws-permissions)
10. [Step 7 — Create the kOps S3 State Store](#step-7--create-the-kops-s3-state-store)
11. [Step 8 — Configure the kOps Environment](#step-8--configure-the-kops-environment)
12. [Step 9 — Generate the Cluster SSH Key](#step-9--generate-the-cluster-ssh-key)
13. Cluster Definition and Provisioning
14. Cluster Validation
15. Node Verification
16. SSH Access
17. Troubleshooting and Lessons Learned
18. Cluster Cleanup
19. Next Steps

# Architecture Overview

The cluster created during this setup has the following high-level architecture:

```text
                         AWS
                          │
                          ▼
                  eu-west-1 Region
                          │
                          ▼
                   eu-west-1a AZ
                          │
                          ▼
                 ohjayy.k8s.local
                          │
                  Kubernetes Cluster
                          │
              ┌───────────┴───────────┐
              │                       │
        Control Plane              Workers
              │                       │
         1 × t3.medium          2 × t3.medium
                                      │
                                ┌─────┴─────┐
                             Worker 1    Worker 2
```

For this deployment:

| Component | Configuration |
|---|---|
| Cloud Provider | AWS |
| Region | `eu-west-1` |
| Availability Zone | `eu-west-1a` |
| Cluster Name | `ohjayy.k8s.local` |
| Control Plane | 1 × `t3.medium` |
| Worker Nodes | 2 × `t3.medium` |
| Networking | Cilium |
| State Store | Amazon S3 |
| Node OS | Ubuntu |
| Container Runtime | containerd |

> [!NOTE]
> `eu-west-1`, `eu-west-1a`, `ohjayy.k8s.local`, and `t3.medium` are the values used for this particular lab. They are **not mandatory kOps values**.
>
> Choose your region, Availability Zones, cluster name, topology, and EC2 instance types according to your own requirements.

## How kOps Builds the Cluster

Before running commands, it is useful to understand the relationship between the tools involved.

```text
                        Administrator
                              │
                              ▼
                       kOps Server
                 ┌────────────┴────────────┐
                 │                         │
               kOps                     kubectl
                 │                         │
                 ▼                         ▼
        Defines / manages           Kubernetes API
        cluster lifecycle                  │
                 │                         │
                 ▼                         ▼
           Amazon S3                Kubernetes Cluster
          State Store
                 │
                 ▼
          Desired Cluster
          Configuration
                 │
                 ▼
                AWS
                 │
        ┌────────┼─────────┐
        │        │         │
       EC2      VPC       IAM
        │        │         │
        └────────┼─────────┘
                 │
                 ▼
          Kubernetes Cluster
```

There are two command-line tools that will be used frequently, but they serve different purposes.

### kOps

`kops` manages the **lifecycle of the Kubernetes cluster itself**.

It is responsible for operations such as:

- defining the desired cluster configuration;
- storing and reading cluster state;
- provisioning the AWS infrastructure;
- creating and managing instance groups;
- updating cluster infrastructure;
- validating the cluster;
- exporting Kubernetes configuration;
- and eventually deleting the cluster when it is no longer required.

### kubectl

`kubectl` communicates with the **Kubernetes API after the cluster exists**.

It is used for operations such as:

- viewing nodes;
- creating Deployments;
- inspecting Pods;
- creating Services;
- checking logs;
- troubleshooting workloads;
- and managing Kubernetes resources.

A useful mental model is:

```text
kOps
 │
 └── Builds and manages the Kubernetes cluster

kubectl
 │
 └── Uses and administers the Kubernetes cluster
```

## Prerequisites

Before beginning, you should have:

- An AWS account
- An Ubuntu EC2 instance to use as the kOps administration server
- SSH access to that server
- AWS permissions sufficient for kOps to provision infrastructure
- Basic Linux command-line knowledge
- Basic familiarity with AWS networking and EC2

The administration server does **not** become one of the Kubernetes nodes.

It is the machine from which kOps and kubectl are executed.

```text
Administration EC2
      │
      │ kOps / kubectl
      ▼
AWS Kubernetes Infrastructure
      │
      ├── Control Plane
      ├── Worker Node
      └── Worker Node
```

> [!WARNING]
> This lab creates billable AWS resources, including EC2 instances and other supporting infrastructure.
>
> Monitor your AWS account while working and clean up resources when they are no longer required.

### Step 1 — Prepare the kOps Administration Server

Launch an Ubuntu EC2 instance that will act as the administration server.

SSH into the server using the key associated with the EC2 instance.

Once connected, update the package index:

```bash
sudo apt update
```

Optionally apply available package upgrades:

```bash
sudo apt upgrade -y
```

You can verify the operating system with:

```bash
cat /etc/os-release
```

This machine will host the tools used throughout the setup:

```text
kOps Administration Server
        │
        ├── AWS CLI
        ├── kOps
        ├── kubectl
        └── SSH credentials
```

> [!NOTE]
> This server is a **management machine**, not the Kubernetes control-plane node.
>
> kOps will later provision separate EC2 instances for the actual Kubernetes cluster.

### Step 2 — Create a Dedicated kOps User

Instead of performing all cluster-management operations from the default Ubuntu account, create a dedicated Linux user called `kops`.

```bash
sudo adduser kops
```

Grant the user passwordless sudo privileges:

```bash
echo "kops ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/kops
```

Switch into the new account:

```bash
sudo su - kops
```

Your shell prompt should now resemble:

```text
kops@kops-server:~$
```

There are two different pieces of information in this prompt:

```text
kops@kops-server
 │       │
 │       └── Linux hostname
 │
 └── Linux username
```

The username is:

```text
kops
```

The hostname may be something such as:

```text
kops-server
```

These are **not the same thing**.

Verify them independently:

```bash
whoami
```

```bash
hostname
```

Example:

```text
$ whoami
kops

$ hostname
kops-server
```

> [!NOTE]
> `kops` is the user and `kops-server` is the machine name. So, from this point onward, perform the remaining kOps setup from the dedicated `kops` account unless a command specifically requires another context.

### Step 3 — Install and Configure AWS CLI

kOps needs access to AWS APIs to create and manage infrastructure.

First check whether AWS CLI is already available:

```bash
aws --version
```

If AWS CLI is not installed, install it using the official AWS CLI installation method appropriate for your system.

After installation, verify it:

```bash
aws --version
```

The command should return the installed AWS CLI version.

## Configure AWS Access

How AWS authentication is configured depends on your environment.

Possible approaches include:

- attaching an IAM role to the administration EC2 instance;
- configuring AWS CLI credentials;
- or using another supported AWS authentication mechanism.

If using locally configured AWS credentials, run:

```bash
aws configure
```

You will be prompted for values such as:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name
Default output format
```

For this deployment, the AWS region used was:

```text
eu-west-1
```

Verify that AWS authentication is working:

```bash
aws sts get-caller-identity
```

A successful response confirms that the AWS CLI can authenticate to AWS.

You can also confirm the configured region:

```bash
aws configure get region
```

> [!WARNING]
> Never commit AWS access keys, secret keys, session tokens, or other credentials to GitHub.
>
> Your README should document **how credentials are configured**, not contain the credentials themselves.

### Step 4 — Install kOps

kOps is the primary cluster lifecycle management tool used in this project.

Download the appropriate kOps binary for your architecture.

For a standard AMD64 Linux machine, a typical installation pattern is:

```bash
curl -Lo kops https://github.com/kubernetes/kops/releases/latest/download/kops-linux-amd64
```

Make the binary executable:

```bash
chmod +x kops
```

Move it into a directory available through the system `PATH`:

```bash
sudo mv kops /usr/local/bin/kops
```

Verify the installation:

```bash
kops version
```

If a version is returned successfully, kOps is installed and accessible.

> [!NOTE]
> Using `latest` is convenient for a lab. For production or reproducible environments, consider explicitly pinning and documenting the kOps version you intend to use.

### Step 5 — Install kubectl

`kubectl` is the Kubernetes command-line client.

While kOps manages the cluster infrastructure and lifecycle, kubectl will be used to communicate with the Kubernetes API after the cluster is running.

Download kubectl:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Make it executable:

```bash
chmod +x kubectl
```

Move it into the system path:

```bash
sudo mv kubectl /usr/local/bin/kubectl
```

Verify the client:

```bash
kubectl version --client
```

At this point the administration server contains the three core interfaces needed for the setup:

```text
                  kOps Server
                      │
          ┌───────────┼───────────┐
          │           │           │
       AWS CLI       kOps       kubectl
          │           │           │
          ▼           ▼           ▼
         AWS      Cluster       Kubernetes
                  Lifecycle        API
```


### Step 6 — Configure AWS Permissions

kOps needs permission to create and manage AWS resources on your behalf.

Depending on the cluster configuration, kOps may interact with services such as:

- Amazon EC2
- Amazon VPC
- IAM
- Elastic Load Balancing
- Amazon S3
- Auto Scaling
- Route tables and Internet Gateways
- Security Groups
- EBS
- SQS
- EventBridge

For a temporary learning environment, broad permissions may be used to simplify the lab.

For production environments, IAM permissions should be designed according to the **principle of least privilege**.

> [!WARNING]
> Broad administrative permissions are convenient for learning but should not automatically be carried into a production design. Review and restrict IAM permissions appropriately before using the same approach in a production environment.

If an IAM role is attached to the administration EC2 instance, verify the effective identity with:

```bash
aws sts get-caller-identity
```

Do not continue until AWS authentication and the required permissions are working.

### Step 7 — Create the kOps S3 State Store

kOps needs persistent storage for the cluster's configuration and state.

For this AWS deployment, an **Amazon S3 bucket** is used as the kOps state store.

The relationship is:

```text
kOps CLI
   │
   ▼
Desired Cluster Configuration
   │
   ▼
Amazon S3 State Store
   │
   ▼
Persistent source of cluster state
```

#### Choose a Bucket Name

S3 bucket names are **globally unique across AWS**.

This means a simple example such as:

```text
proc-kops
```

may already belong to another AWS customer.

Choose a unique bucket name.

For example:

```text
your-unique-kops-state-store
```

The bucket used during this lab was:

```text
ohjayy
```

> [!IMPORTANT]
> `ohjayy` is an example from this deployment.
>
> Do not assume it will be available for your account. Create your own globally unique S3 bucket name.

Create the bucket in your chosen region:

```bash
aws s3 mb s3://<YOUR-UNIQUE-BUCKET-NAME> --region <YOUR-AWS-REGION>
```

For example:

```bash
aws s3 mb s3://ohjayy --region eu-west-1
```

A successful result resembles:

```text
make_bucket: ohjayy
```

You can verify the bucket exists with:

```bash
aws s3 ls
```

### Step 8 — Configure the kOps Environment

Two environment variables make subsequent kOps commands considerably easier:

- `NAME` — the Kubernetes cluster name
- `KOPS_STATE_STORE` — the S3 location where kOps stores cluster state

For this deployment:

```bash
export NAME=ohjayy.k8s.local
export KOPS_STATE_STORE=s3://ohjayy
```

However, typing these manually every time a new shell session starts is inconvenient.

Add them to the dedicated `kops` user's `~/.bashrc` file.

Open the file:

```bash
vi ~/.bashrc
```

Add:

```bash
# kOps cluster configuration
export NAME=ohjayy.k8s.local
export KOPS_STATE_STORE=s3://ohjayy
```

> [!IMPORTANT]
> `ohjayy.k8s.local` is the cluster name used for **this deployment**. The `ohjayy` portion is the name I chose for my cluster; it is not a required kOps value. When following this guide, replace it with your own cluster name. For example:
>
> ```bash
> export NAME=<YOUR-CLUSTER-NAME>.k8s.local
> ```
>
> Likewise, replace `s3://ohjayy` with the S3 state-store bucket you created in the previous step.

Save the file and reload it:

```bash
source ~/.bashrc
```

Verify both variables:

```bash
echo $NAME
```

```bash
echo $KOPS_STATE_STORE
```

Expected output for this deployment:

```text
ohjayy.k8s.local
s3://ohjayy
```

The relationship now looks like this:

```text
$NAME
  │
  └── ohjayy.k8s.local
           │
           ▼
    Kubernetes Cluster

$KOPS_STATE_STORE
  │
  └── s3://ohjayy
           │
           ▼
      kOps State
```

> [!IMPORTANT]
> Add these variables to the `.bashrc` belonging to the user that will actually run kOps. If you are working as:
>
> ```text
> kops@kops-server
> ```
>
> the relevant file is:
>
> ```text
> /home/kops/.bashrc
> ```
>
> Adding the variables to another user's `.bashrc` will not automatically make them available to the `kops` account.

#### About the `.k8s.local` Cluster Name

This lab uses:

```text
ohjayy.k8s.local
```

Some older kOps guides describe `.k8s.local` clusters as using **gossip-based DNS** and therefore not requiring a Route 53 hosted zone.

With the kOps version used during this deployment, cluster creation instead produced the warning:

```text
Gossip is deprecated, using None DNS instead
```

Therefore, do not assume that older descriptions of `.k8s.local` gossip behaviour exactly match the behaviour of current kOps releases.

> [!NOTE]
> kOps behaviour changes between releases.
>
> Always check the output produced by your installed kOps version rather than relying entirely on an older tutorial.

### Step 9 — Generate the Cluster SSH Key

kOps can install an SSH public key on the EC2 instances it creates.

This allows the administrator to SSH into Kubernetes nodes later for inspection or troubleshooting.

Generate an Ed25519 key pair from the `kops` account:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

This creates:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

Their roles are different:

```text
id_ed25519
     │
     └── PRIVATE KEY
         Remains on the administration server

id_ed25519.pub
     │
     └── PUBLIC KEY
         Registered with kOps and installed on cluster nodes
```

The private key should never be committed to Git.

Verify the files:

```bash
ls -la ~/.ssh/
```

You should see both the private and public key.

> [!WARNING]
> Never publish:
>
> ```text
> ~/.ssh/id_ed25519
> ```
>
> The `.pub` file is the public portion of the key. The file without `.pub` is the private key and must remain protected.

At this stage, the administration environment is ready:

```text
                       kOps Server
                           │
          ┌────────────────┼────────────────┐
          │                │                │
       AWS CLI            kOps           kubectl
          │                │                │
          │                │                │
          └──────────┬─────┴────────────────┘
                     │
             Environment Variables
                     │
          ┌──────────┴──────────┐
          │                     │
   NAME=ohjayy.k8s.local   KOPS_STATE_STORE
                           =s3://ohjayy
          │                     │
          └──────────┬──────────┘
                     │
                     ▼
                SSH Key Pair
                     │
                     ▼
             Ready to Define
            Kubernetes Cluster
```

## Part 1 Checkpoint

Before moving to cluster creation, verify the following:

```bash
whoami
```

Expected:

```text
kops
```

Verify AWS access:

```bash
aws sts get-caller-identity
```

Verify kOps:

```bash
kops version
```

Verify kubectl:

```bash
kubectl version --client
```

Verify the cluster environment:

```bash
echo $NAME
echo $KOPS_STATE_STORE
```

Verify the SSH key:

```bash
ls -l ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

At this point:

```text
Ubuntu Administration Server       ✓
Dedicated kOps User                ✓
AWS CLI                            ✓
AWS Authentication                 ✓
kOps                               ✓
kubectl                            ✓
IAM Permissions                    ✓
S3 State Store                     ✓
Cluster Environment Variables      ✓
SSH Key Pair                       ✓
                                    │
                                    ▼
                       Ready for Cluster Creation
```

## Cluster Definition and Provisioning

With the administration environment prepared, the next stage is to define your Kubernetes cluster, store its desired configuration in the kOps state store, register the SSH public key, and allow kOps to provision the required AWS infrastructure.

The important distinction throughout this section is:

```text
Define the cluster
      │
      ▼
Store desired configuration in S3
      │
      │    No Kubernetes cluster yet
      │
      ▼
Review the desired infrastructure
      │
      ▼
Apply the configuration
      │
      ▼
Provision AWS resources
      │
      ▼
Bootstrap Kubernetes
```

Creating the cluster definition and actually provisioning the cluster are **separate operations** in kOps.

### Step 10 — Confirm the AWS Availability Zone

Before defining your cluster, confirm which Availability Zones are available in your chosen AWS region.

For this deployment, the region used was:

```text
eu-west-1
```

To check the Availability Zones available in your region, run:

```bash
aws ec2 describe-availability-zones \
  --region eu-west-1 \
  --query 'AvailabilityZones[].ZoneName' \
  --output table
```

For this deployment, AWS returned:

```text
eu-west-1a
eu-west-1b
eu-west-1c
```

The cluster created in this guide uses:

```text
eu-west-1a
```

> [!IMPORTANT]
> Do not blindly copy `eu-west-1a`.
>
> Check the Availability Zones available in **your own AWS region** and choose the appropriate zone for your cluster.

If your region is different, replace the region in the command:

```bash
aws ec2 describe-availability-zones \
  --region <YOUR-AWS-REGION> \
  --query 'AvailabilityZones[].ZoneName' \
  --output table
```

For example, if you are deploying in another region, AWS may return completely different Availability Zone names.

The topology used for this deployment is:

```text
                     eu-west-1
                         │
                         ▼
                    eu-west-1a
                         │
                         ▼
                ohjayy.k8s.local
                         │
             ┌───────────┴───────────┐
             │                       │
       Control Plane              Workers
             │                       │
        1 × t3.medium          2 × t3.medium
```

### Step 11 — Define the Kubernetes Cluster

Now define your desired Kubernetes cluster using kOps.

For this deployment, the command used was:

```bash
kops create cluster \
  --name=${NAME} \
  --zones=eu-west-1a \
  --networking=cilium \
  --control-plane-size=t3.medium \
  --control-plane-count=1 \
  --node-size=t3.medium \
  --node-count=2
```

Let's break down what this command defines.

| Option | Purpose |
|---|---|
| `--name=${NAME}` | Uses the cluster name stored in your `NAME` environment variable |
| `--zones=eu-west-1a` | Places the cluster in the selected AWS Availability Zone |
| `--networking=cilium` | Configures Cilium as the Kubernetes networking solution |
| `--control-plane-size=t3.medium` | Uses `t3.medium` for the control-plane EC2 instance |
| `--control-plane-count=1` | Creates one control-plane instance |
| `--node-size=t3.medium` | Uses `t3.medium` for worker EC2 instances |
| `--node-count=2` | Defines two worker nodes |

> [!IMPORTANT]
> These values describe **this lab**, not mandatory kOps requirements.
>
> In particular, `t3.medium`, one control-plane node, two worker nodes, `eu-west-1a`, and Cilium were choices made for this deployment.
>
> Select instance types, node counts, networking, regions, and Availability Zones appropriate for your own environment.

In this project, `$NAME` was configured earlier as:

```bash
export NAME=ohjayy.k8s.local
```

Therefore:

```bash
--name=${NAME}
```

resolves to:

```text
--name=ohjayy.k8s.local
```

`ohjayy.k8s.local` is the cluster name used for this project. When reproducing the setup, use the cluster name you configured in your own `NAME` environment variable.

#### What `kops create cluster` Actually Does

This is one of the most important concepts in the entire setup.

When you run:

```bash
kops create cluster ...
```

kOps does **not immediately create the EC2 instances and complete Kubernetes cluster**.

Instead, kOps calculates your desired cluster configuration and writes that desired state to the configured state store.

In this project:

```text
KOPS_STATE_STORE=s3://ohjayy
```

and:

```text
NAME=ohjayy.k8s.local
```

Therefore, the cluster configuration is maintained under the kOps state stored in Amazon S3.

Conceptually:

```text
kops create cluster
        │
        ▼
Read cluster parameters
        │
        ▼
Generate desired configuration
        │
        ▼
Store configuration/state in S3
        │
        ▼
Your S3 State Store
        │
        ▼
Your Cluster State
        │
        │
        └── Desired cluster exists
            but AWS cluster infrastructure
            has not yet been applied
```

This distinction becomes important when working with infrastructure automation:

```text
DESIRED STATE ≠ RUNNING INFRASTRUCTURE
```

The command defines what kOps **should build**.

A later command tells kOps to actually apply that definition to AWS.

#### Expected Creation Output

kOps may print a large list of tasks and resources it intends to manage.

Near the end of the output, you should see messages similar to:

```text
Must specify --yes to apply changes
Cluster configuration has been created.
```

That is expected.

It means:

```text
Cluster Definition       ✓
State Stored              ✓
AWS Infrastructure       Not applied yet
```

Depending on the kOps version you are using, you may also see:

```text
Gossip is deprecated, using None DNS instead
```

This warning appeared during this deployment. It relates to the `.k8s.local` naming behaviour discussed earlier and is another reason not to assume that older kOps tutorials exactly reflect current releases.

### Step 12 — Understand What kOps Plans to Create

The output from `kops create cluster` may be lengthy because a Kubernetes cluster requires considerably more than three EC2 instances.

Although your visible Kubernetes topology may be:

```text
1 Control Plane
       +
2 Worker Nodes
```

kOps must build supporting AWS infrastructure around those machines.

Depending on your kOps version and cluster configuration, the resulting environment can include resources such as:

```text
                         AWS
                          │
                          ▼
                         VPC
                          │
            ┌─────────────┼─────────────┐
            │             │             │
         Subnet      Route Tables   Internet Gateway
            │
            ▼
     Security Groups
            │
            ▼
       EC2 Instances
       ┌─────┴─────┐
       │           │
Control Plane    Workers
       │
       └─────┬─────┘
             │
       Auto Scaling Groups
             │
             ▼
      Load Balancer / NLB
             │
             ▼
       Kubernetes API

Additional supporting resources:
IAM Roles / Instance Profiles
EBS Volumes
Target Groups
SQS / EventBridge resources
and other kOps-managed components
```

The exact resource set may vary depending on your kOps version and cluster configuration.

> [!NOTE]
> This is why a kOps cluster should not be thought of as simply "three EC2 instances."
>
> kOps builds and manages the surrounding AWS infrastructure required for those instances to operate as a Kubernetes cluster.

### Step 13 — Register the SSH Public Key

Before provisioning your cluster, register the SSH public key created earlier.

Run:

```bash
kops create secret \
  --name ${NAME} \
  sshpublickey admin \
  -i ~/.ssh/id_ed25519.pub
```

This tells kOps to associate the public portion of your Ed25519 key with the cluster.

Remember:

```text
~/.ssh/id_ed25519
        │
        └── Private key
            Keep on your administration server

~/.ssh/id_ed25519.pub
        │
        └── Public key
            Register with kOps
```

The purpose is to enable this later:

```text
kOps Administration Server
          │
          │ Private Key
          │ ~/.ssh/id_ed25519
          │
          ▼
    Kubernetes EC2 Node
          ▲
          │
          │ Matching Public Key
          │ installed during provisioning
```

The command may return to your shell without producing significant output. That does not necessarily indicate a problem.

At this stage:

```text
Cluster Desired State     ✓
SSH Public Key            ✓
AWS Infrastructure        Not yet applied
```

### Step 14 — Provision the Cluster

Now instruct kOps to apply your desired configuration to AWS:

```bash
kops update cluster ${NAME} --yes --admin
```

This is the command that moves the setup from **configuration** to **real infrastructure**.

The difference between the two major commands is:

```text
kops create cluster
        │
        ▼
Define desired cluster configuration
        │
        ▼
Store configuration/state in S3
        │
        │
        │   No running cluster yet
        │
        ▼
kops update cluster ${NAME} --yes --admin
        │
        ▼
Apply desired configuration
        │
        ▼
Provision AWS infrastructure
        │
        ▼
Bootstrap Kubernetes
```

The `--yes` flag is especially important.

Without it, kOps can calculate and display changes without applying them.

With:

```text
--yes
```

you are authorising kOps to make the required changes in AWS.

The `--admin` option also enables administrative Kubernetes access as part of the workflow.

> [!WARNING]
> This is the point where kOps begins creating billable AWS infrastructure. Review your intended cluster configuration before applying it.

#### What to Expect During Provisioning

During your deployment, kOps will begin creating and configuring the AWS infrastructure required by the cluster.

Depending on your configuration, this may include resources for areas such as:

```text
Networking
   │
   ├── VPC
   ├── Subnet
   ├── Routes
   └── Internet Gateway

Compute
   │
   ├── Control-plane EC2
   └── Worker EC2 instances

Security / Identity
   │
   ├── IAM roles
   ├── Instance profiles
   └── Security groups

Scaling
   │
   ├── Control-plane instance group
   └── Worker instance group

Kubernetes API Access
   │
   ├── Network Load Balancer
   └── Target infrastructure
```

When the provisioning tasks complete, kOps may report that your cluster is starting and will require several minutes to become ready.

This is normal.

Creating the AWS infrastructure and having a fully functioning Kubernetes cluster are not instantaneous operations.

### Step 15 — Export the Kubernetes Configuration

`kubectl` needs connection information and credentials before it can communicate with your new Kubernetes API.

This information is stored in a **kubeconfig**.

Depending on your kOps version, `kops update cluster ${NAME} --yes --admin` may already report:

```text
Exporting kubeconfig for cluster
```

followed by a message similar to:

```text
kOps has set your kubectl context to <YOUR-CLUSTER-NAME>
```

During this deployment, kOps automatically performed that export. However, the kubeconfig was also explicitly exported afterwards:

```bash
kops export kubecfg ${NAME} --admin
```

A successful result should resemble:

```text
kOps has set your kubectl context to <YOUR-CLUSTER-NAME>
```

For this deployment, that cluster name was:

```text
ohjayy.k8s.local
```

The relationship is:

```text
kOps State
    │
    ▼
kops export kubecfg
    │
    ▼
Local kubeconfig
    │
    ▼
kubectl
    │
    ▼
Kubernetes API
```

Without the correct kubeconfig, simply having `kubectl` installed does not tell it:

- which Kubernetes API server to contact;
- which cluster to use;
- which credentials to present;
- or which context should be active.

You can inspect your currently selected context with:

```bash
kubectl config current-context
```

The output should correspond to the cluster name configured in your `NAME` variable.

For this deployment, that is:

```text
ohjayy.k8s.local
```

> [!NOTE]
> Explicitly running `kops export kubecfg ${NAME} --admin` makes the kubeconfig step clear and reproducible even when the preceding kOps command has already performed an export.

## Cluster Validation

Provisioning completing successfully does not automatically mean every Kubernetes component will immediately be healthy.

Your newly created instances still need time to:

- boot;
- initialise Kubernetes services;
- establish networking;
- join the cluster;
- start system Pods;
- and become Ready.

### Step 16 — Validate the Cluster

Run:

```bash
kops validate cluster --wait 10m
```

The `--wait 10m` option is useful because your newly provisioned cluster may not be healthy on the first validation attempt.

Instead of immediately giving up, kOps continues checking your cluster for up to ten minutes.

Conceptually:

```text
kops create cluster
        │
        ▼
Desired configuration stored
        │
        ▼
SSH public key registered
        │
        ▼
kops update cluster --yes --admin
        │
        ▼
AWS infrastructure provisioned
        │
        ▼
Cluster bootstrapping
        │
        ▼
kops validate cluster --wait 10m
        │
        ▼
     Healthy?
       /   \
     No     Yes
     │       │
     ▼       ▼
Retry      Cluster Ready
until
timeout
```

#### Initial Validation Messages

During your deployment, the first validation checks may **not** immediately report a healthy cluster.

Initially:

- the control-plane node may appear but will not yet be Ready;
- the worker machines may not have completely joined;
- and several Kubernetes system Pods may still be pending.

As your cluster continues bootstrapping, you may see a progression similar to:

```text
Stage 1
Control Plane      Not Ready
Worker Nodes       Joining
System Pods        Pending

        │
        ▼

Stage 2
Control Plane      Ready
Worker Nodes       Joining / Not Ready
System Pods        Starting

        │
        ▼

Stage 3
Control Plane      Ready
Worker Nodes       Ready
System Pods        Completing startup

        │
        ▼

Stage 4
Cluster            Ready
```

The exact sequence and timing may differ in your environment.

> [!NOTE]
> An initial validation failure immediately after provisioning does **not necessarily mean that your cluster creation failed**. Kubernetes nodes and system components need time to bootstrap. Using:
>
> ```bash
> kops validate cluster --wait 10m
> ```
>
> allows kOps to continue checking while your cluster becomes healthy.

Once bootstrapping is complete, you should eventually receive a message similar to:

```text
Your cluster <YOUR-CLUSTER-NAME> is ready
```

For this deployment, the final message was:

```text
Your cluster ohjayy.k8s.local is ready
```

That confirms that kOps considers the cluster healthy.

### Step 17 — Review the Instance Groups

During validation, kOps reports your cluster's instance groups.

For the topology used in this guide, they will conceptually resemble:

```text
INSTANCE GROUP                 ROLE            MACHINE TYPE   COUNT   ZONE

control-plane-eu-west-1a       ControlPlane    t3.medium      1       eu-west-1a

nodes-eu-west-1a               Node            t3.medium      2       eu-west-1a
```

Your names, instance types, counts, and Availability Zones may differ according to your configuration.

This introduces an important kOps concept: **Instance Groups**.

kOps groups machines according to their role and configuration.

For this deployment:

```text
                    ohjayy.k8s.local
                           │
             ┌─────────────┴─────────────┐
             │                           │
      kOps Instance Group         kOps Instance Group
   control-plane-eu-west-1a        nodes-eu-west-1a
             │                           │
       1 × t3.medium                2 × t3.medium
             │                           │
       Control Plane              Worker Nodes
```

Your control-plane machine will belong to its control-plane instance group, while your workers will belong to the worker instance group.

This is different from simply thinking of the machines as independent EC2 instances. kOps manages them as members of defined infrastructure groups.

## Node Verification

Once kOps reports that your cluster is healthy, use `kubectl` to verify what Kubernetes itself sees.

### Step 18 — Check the Kubernetes Nodes

Run:

```bash
kubectl get nodes
```

Your output should show the control-plane and worker nodes together with their current status.

For this deployment, the result was:

```text
NAME                  STATUS   ROLES           AGE     VERSION
i-0399821111d4ae656   Ready    node            7m59s   v1.36.4
i-0bdec6e1eb77659a1   Ready    control-plane   8m53s   v1.36.4
i-0f70191995273bfb2   Ready    node            8m      v1.36.4
```

Your exact instance IDs, ages, and Kubernetes version will likely differ.

For the three-node topology used in this guide, what you want to see is:

```text
1 × control-plane   Ready
2 × node            Ready
```

Depending on your AWS/kOps configuration, the node names may correspond to AWS EC2 instance IDs.

For example:

```text
AWS EC2 Instance
        │
        │ Instance ID
        ▼
i-0bdec6e1eb77659a1
        │
        ▼
Kubernetes Node
        │
        ▼
control-plane
```

This makes it possible to relate the Kubernetes node back to the underlying AWS EC2 machine.

#### What `Ready` Means

When your node shows:

```text
STATUS
Ready
```

Kubernetes considers that node healthy and available to perform its assigned role.

For the topology used in this guide, a healthy result should therefore look conceptually like:

```text
Control Plane     Ready ✓
Worker 1          Ready ✓
Worker 2          Ready ✓
```

At this point, your basic three-node Kubernetes topology is operational.

### Step 19 — Get a More Detailed Node View

For a cleaner but more detailed operational view, run:

```bash
kubectl get nodes -o wide
```

This provides additional information including:

- internal/private IP addresses;
- external/public IP addresses;
- operating system;
- kernel;
- container runtime;
- Kubernetes version;
- node role.

For this sample deployment, the output was:

```text
NAME                  STATUS   ROLES           AGE   VERSION   INTERNAL-IP      EXTERNAL-IP     OS-IMAGE             KERNEL-VERSION            CONTAINER-RUNTIME
i-0399821111d4ae656   Ready    node            10m   v1.36.4   172.20.87.214    3.252.93.139    Ubuntu 24.04.4 LTS   6.17.0-1019-aws (amd64)   containerd://2.2.4
i-0bdec6e1eb77659a1   Ready    control-plane   11m   v1.36.4   172.20.150.216   3.248.219.14    Ubuntu 24.04.4 LTS   6.17.0-1019-aws (amd64)   containerd://2.2.4
i-0f70191995273bfb2   Ready    node            10m   v1.36.4   172.20.231.96    3.249.197.192   Ubuntu 24.04.4 LTS   6.17.0-1019-aws (amd64)   containerd://2.2.4
```

This particular deployment confirms:

```text
Kubernetes Version     v1.36.4
Node OS                Ubuntu 24.04.4 LTS
Container Runtime      containerd 2.2.4
Instance Count         3
All Nodes              Ready
```

> [!NOTE]
> IP addresses, EC2 instance IDs, Kubernetes versions, kernel versions, and software versions shown above are snapshots from this specific deployment.
>
> Your output will differ.

For normal cluster verification, `-o wide` is often more readable than displaying every Kubernetes node label.

### Step 20 — Inspect Node Labels

Kubernetes attaches metadata called **labels** to your nodes.

To inspect them:

```bash
kubectl get nodes --show-labels
```

This can produce a very long output because every label is printed in a single column.

Depending on your environment, useful labels may include:

```text
node-role.kubernetes.io/control-plane=
node-role.kubernetes.io/node=
node.kubernetes.io/instance-type=t3.medium
topology.kubernetes.io/region=eu-west-1
topology.kubernetes.io/zone=eu-west-1a
kops.k8s.io/instancegroup=control-plane-eu-west-1a
kops.k8s.io/instancegroup=nodes-eu-west-1a
kubernetes.io/os=linux
kubernetes.io/arch=amd64
```

These labels describe properties of your nodes.

For example:

```text
node.kubernetes.io/instance-type=t3.medium
                  │
                  └── AWS EC2 instance type

topology.kubernetes.io/region=eu-west-1
                  │
                  └── AWS region

topology.kubernetes.io/zone=eu-west-1a
                  │
                  └── AWS Availability Zone

kops.k8s.io/instancegroup=nodes-eu-west-1a
                  │
                  └── kOps instance group
```

Labels become especially important later for concepts such as:

- node selection;
- workload scheduling;
- topology awareness;
- grouping resources;
- affinity and anti-affinity;
- and policy decisions.

> [!TIP]
> `kubectl get nodes --show-labels` is useful when you specifically need to inspect node metadata, but the output can be lengthy. For a normal operational overview, use:
>
> ```bash
> kubectl get nodes -o wide
> ```

At this stage, both kOps and Kubernetes should have independently confirmed that your cluster is operational:

```text
                kOps
                 │
                 │ validate cluster
                 ▼
          Cluster Healthy ✓
                 │
                 ▼
              kubectl
                 │
                 │ get nodes
                 ▼
        ┌────────┼────────┐
        │        │        │
     Worker   Control   Worker
      Ready    Ready     Ready
```

## SSH Access

With the cluster validated and all three nodes reporting `Ready`, the final setup check is direct SSH access to one of the Kubernetes nodes.

This is not required for normal Kubernetes workload management, which should generally be performed through `kubectl`, but it can be useful for node-level inspection and troubleshooting.

### Step 21 — SSH into the Control-Plane Node

From the previous `kubectl get nodes -o wide` output, identify the public IP address of the node you want to access.

The general command is:

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<NODE-PUBLIC-IP>
```

For this deployment, the control-plane node had the public IP:

```text
3.248.219.14
```

Therefore, the command used was:

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@3.248.219.14
```

On your first connection, SSH may display a message similar to:

```text
The authenticity of host '<IP-ADDRESS>' can't be established.
ED25519 key fingerprint is SHA256:<FINGERPRINT>.

Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

After confirming the host, SSH adds it to the `known_hosts` file and continues the connection.

A successful login should change your prompt from the kOps administration server:

```text
kops@kops-server:~$
```

to the Kubernetes node:

```text
ubuntu@<EC2-INSTANCE-ID>:~$
```

For this deployment:

```text
ubuntu@i-0bdec6e1eb77659a1:~$
```

This confirms that the SSH public key registered with kOps was successfully installed on the node and that the corresponding private key can authenticate to it.

> [!IMPORTANT]
> kOps may generate an SSH suggestion that references:
>
> ```text
> ~/.ssh/id_rsa
> ```
>
> Do not blindly use that path. This guide created an Ed25519 key, so the correct private key for this deployment is:
>
> ```text
> ~/.ssh/id_ed25519
> ```
>
> Always use the key that actually exists in your environment.

When you have finished working on the node, exit the SSH session:

```bash
exit
```

You should return to:

```text
kops@kops-server:~$
```

## Troubleshooting and Lessons Learned

Most of the important behaviours encountered during this setup have already been explained at the relevant stages of the guide. The following points are worth keeping in mind when reproducing the deployment.

### Always Use Your Own Environment Values

Commands in this guide contain values from this particular deployment, including:

```text
ohjayy.k8s.local
s3://ohjayy
eu-west-1
eu-west-1a
t3.medium
```

These values are examples, not requirements.

Where appropriate, substitute your own:

```text
<YOUR-CLUSTER-NAME>
<YOUR-S3-BUCKET>
<YOUR-AWS-REGION>
<YOUR-AVAILABILITY-ZONE>
<YOUR-INSTANCE-TYPE>
```

The same applies to dynamically generated values such as:

```text
EC2 instance IDs
Public IP addresses
Private IP addresses
Kubernetes versions
```

### Check the Current Shell Context

Several operations in this guide depend on being executed from the dedicated `kops` account.

Before troubleshooting missing environment variables, SSH keys, or configuration files, confirm your current user:

```bash
whoami
```

For this setup:

```text
kops
```

Your prompt should normally resemble:

```text
kops@kops-server:~$
```

### Trust the Output of Your Installed kOps Version

kOps behaviour can change between releases.

For example, this deployment produced:

```text
Gossip is deprecated, using None DNS instead
```

even though older tutorials may describe `.k8s.local` clusters differently.

When your installed version behaves differently from an older guide, inspect the current command output and documentation rather than assuming the older behaviour still applies.

### Do Not Treat Silent Commands as Automatic Failures

Some commands may complete successfully without producing substantial output.

For example, registering the SSH public key:

```bash
kops create secret \
  --name ${NAME} \
  sshpublickey admin \
  -i ~/.ssh/id_ed25519.pub
```

may simply return to the shell prompt.

When this happens, continue with the relevant verification step rather than assuming that silence means failure.

## Cluster Cleanup

The cluster created in this guide is being retained for subsequent Kubernetes workload testing, so cluster deletion was **not performed** as part of this project.

However, when you no longer need your own cluster, kOps provides a deletion workflow.

First, preview the resources that would be removed:

```bash
kops delete cluster ${NAME}
```

When you are certain that the cluster is no longer required:

```bash
kops delete cluster ${NAME} --yes
```

> [!WARNING]
> The `--yes` flag authorises kOps to begin deleting the cluster infrastructure.
>
> Do not run this command if you intend to continue using your cluster.

> [!IMPORTANT]
> Running Kubernetes infrastructure on AWS can generate charges. If you are finished with a lab environment, verify your AWS resources after cleanup and ensure that resources you no longer require are not left running.

The S3 state-store bucket is a separate resource. Do not assume that deleting the cluster automatically means you should delete the bucket.

If the bucket contains state that you still need, retain it. Remove it only when you are certain it is no longer required.

## Next Steps

At this point, the **kOps cluster setup is complete**.

The cluster has been:

```text
Defined
   │
   ▼
Provisioned on AWS
   │
   ▼
Bootstrapped
   │
   ▼
Validated
   │
   ▼
Verified with kubectl
   │
   ▼
Verified through SSH
   │
   ▼
Ready for Workloads
```

The cluster created during this project will remain running for the next stage rather than being deleted.

The next practical exercise will use the existing cluster to deploy a **single containerised microservice**.

This moves the project from Kubernetes infrastructure provisioning into workload deployment:

```text
Existing Kubernetes Cluster
            │
            ▼
    Containerised Application
            │
            ▼
     Kubernetes Deployment
            │
            ▼
            Pods
            │
            ▼
     Kubernetes Service
            │
            ▼
      Application Access
```

This provides a smaller environment for practising Kubernetes application deployment before applying the same concepts to a larger capstone project.

## Conclusion

You have now provisioned a working Kubernetes cluster on AWS using kOps and verified that the control plane and worker nodes are operational.

More importantly, the setup established the complete relationship between the major components used throughout the project:

```text
AWS
 │
 └── Provides the infrastructure

kOps
 │
 └── Creates and manages the Kubernetes cluster

Amazon S3
 │
 └── Stores the kOps cluster state

kubectl
 │
 └── Communicates with the Kubernetes API

Kubernetes
 │
 └── Provides the platform where workloads will run
```

With the infrastructure layer complete, the cluster is now ready for application deployment.
