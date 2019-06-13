# A Walkthrough of Deploying Azure Simple Bedrock Environment

This document walks through the necessary steps to deploy an Bedrock deployment using the 
[Azure Simple](https://github.com/microsoft/bedrock/cluster/environments/azure-simple) environment.  This document will not include the whole of the [gitops](https://github.com/microsoft/bedrock/gitops) workflow.  Instead, this document assumes a pre-existing [Flux Manifest repository](https://github.com/microsoft/bedrock/tree/master/cluster/common/flux) which will be cloned and set up for the needs of this walkthrough.

This walkthrough consists of the following:

- [Setting up Prerequisites](#prerequisites)
- [Configure Terraform For Azure Access](#configure-terraform-for-azure-access)
- [Clone The Bedrock Repository](#clone-the-bedrock-repository)
- [Setup Terraform Deployment Variables for Azure Simple](#setup-terraform-deployment-variables-for-azure-simple)
- [Deploy the Azure Simple Template](#deploy-the-azure-simple-template)
- [Interact with the Deployed Cluster](#interact-with-the-deployed-cluster)

## Prerequisites

Prior to getting started with the deployment, there are a couple of required steps:

1. The required common tools (kubectl, helm, and terraform) need to be installed.  That process is talked about [here](https://github.com/microsoft/bedrock/tree/master/cluster).  
2. The [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) needs to be installed.
3. Cloning and setting up the Flux manifest repository
4. Creating an [Azure Service Principal](https://github.com/microsoft/bedrock/tree/master/cluster/azure/service-principal)
5. Creating an RSA key for logging into AKS nodes

Once the above is completed, we walk through the process of configuring Terraform and the Bedrock scripts, deploy the cluster and check on the deployed cluster's health.

### Installing the required tooling

This document assumes one is running a current version of Ubuntu.  If running some other OS, the steps for installing the software below may differ.

Per [the Bedrock documentation](https://github.com/microsoft/bedrock/tree/master/cluster#required-tools), one must install `helm`, `terraform` and `kubectl`.  The links for each are in the documentation link.

A set of scripts to help with this process can be found [here](https://github.com/jmspring/bedrock-dev-env/tree/master/scripts).  These scripts (which were used for this document) install the tools into `/usr/local/bin`.  In this case, one would want to use `setup_kubernetes_tools.sh` and `setup_terraform.sh`.

### Installing the Azure CLI

The Azure CLI install guide can be found [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).  One can also use this [script](https://github.com/jmspring/bedrock-dev-env/blob/master/scripts/setup_azure_cli.sh) to do so (if running on a Unix based machine).

### Cloning and Setting Up a Flux Manifest Repository

As previously mentioned, this document will leverage a pre-existing Flux manifest repository.  However, there is still a bit of work necessary to do so.  The repository we will base this work off of is [here](https://github.com/andrebriggs/sample_app_manifests/tree/master/prod).  To use this repository we must:

1. Create an RSA keypair for the Flux repository
2. Fork the repository
3. Add the keypair to the repository

#### Create an RSA Key Pair for a Deploy Key for the Flux Repository

The [deploy key](https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys) is generated using `ssk-keygen`.  The public portion will be installed as a deploy key once the repository is forked.

To generate the key:

```bash
kudzu:azure-simple jmspring$ ssh-keygen -b 4096 -t rsa -f ~/.ssh/azure-simple-deploy-key
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /Users/jmspring/.ssh/azure-simple-deploy-key.
Your public key has been saved in /Users/jmspring/.ssh/azure-simple-deploy-key.pub.
The key fingerprint is:
SHA256:jago9v63j05u9WoiNExnPM2KAWBk1eTHT2AmhIWPIXM jmspring@kudzu.local
The key's randomart image is:
+---[RSA 4096]----+
|.=o.B= +         |
|oo E..= .        |
|  + =..oo.       |
|   . +.*o=       |
|    o * S..      |
|   . * . .       |
|... o ... .      |
|...  .o+.. .     |
|  .o..===o.      |
+----[SHA256]-----+
kudzu:azure-simple jmspring$ 
```

We will revisit the key in third step below.

#### Forking the Repository

To fork the repository, click [here](https://github.com/andrebriggs/sample_app_manifests/tree/master/prod).  You should see:

![initial repository](https://raw.githubusercontent.com/jmspring/bedrock-tutorials/master/walkthroughs/azure-simple/images/initial_repository.png)

Click "Fork" and you should see something resembling:

![forked repository](https://raw.githubusercontent.com/jmspring/bedrock-tutorials/master/walkthroughs/azure-simple/images/forked_repository.png)

#### Adding Repository Key

Previously an [RSA key pair was created](#create-an-rsa-key-pair-for-a-deploy-key-for-the-flux-repository).  The public key needs to be added as a deploy key.  First, display the contents of the public key:

```bash
kudzu:azure-simple jmspring$ more ~/.ssh/azure-simple-deploy-key.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDTNdGpnmztWRa8RofHl8dIGyNkEayNR6d7p2JtJ7+zMj0HRUJRc+DWvBML4DvT29AumVEuz1bsVyVS2f611NBmXHHKkbzAZZzv9gt2uB5sjnmm7LAORJyoBEodR/T07hWr8MDzYrGo5fdTDVagpoHcEke6JT04AL21vysBgqfLrkrtcgaXsw8e3rkfbqGLbhb6o1muGdEyE+uci4hRVj+FGL9twh3Mb6+0uak/UsTFgfDi/oTXdXOFIitQ1o40Eip6P4xejEOuIye0cg7rfX461NmOP7HIEsUa+BwMExiXXsbxj6Z0TXG0qZaQXWjvZF+MfHx/J0Alb9kdO3pYx3rJbzmdNFwbWM4I/zN+ng4TFiHBWRxRFmqJmKZX6ggJvX/d3z0zvJnvSmOQz9TLOT4lqZ/M1sARtABPGwFLAvPHAkXYnex0v93HUrEi7g9EnM+4dsGU8/6gx0XZUdH17WZ1dbEP7VQwDPnWCaZ/aaG7BsoJj3VnDlFP0QytgVweWr0J1ToTRQQZDfWdeSBvoqq/t33yYhjNA82fs+bR/1MukN0dCWMi7MqIs2t3TKYW635E7VHp++G1DR6w6LoTu1alpAlB7d9qiq7o1c4N+gakXSUkkHL8OQbQBeLeTG1XtYa//A5gnAxLSzxAgBpVW15QywFgJlPk0HEVkOlVd4GzUw== jmspring@kudzu.local
```

Next, on the newly forked repository, select `Settings` -> `Deploy Keys` -> `Add deploy key`.  Next, give your key a title and paste in the contents of your public key.  Also, allow the key to have `Write Access`.  When complete the screen should resemble:

![enter key](https://raw.githubusercontent.com/jmspring/bedrock-tutorials/master/walkthroughs/azure-simple/images/enter_key.png)

Click "Approve" and one should see:

![approve key](https://raw.githubusercontent.com/jmspring/bedrock-tutorials/master/walkthroughs/azure-simple/images/approve_key.png)

### Creating an Azure Service Principal

We will be using a single Azure Service Principal for both configuring Terraform and for use in the AKS cluster being deployed.  In Bedrock, creating a Service Principal is documented [here](https://github.com/microsoft/bedrock/tree/master/cluster/azure#create-an-azure-service-principal).  

To create a Service Principal, one must [login to the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli).  First, the Id of the subscription is needed by running `az account show`.  Then the Service Principa is created.  The proces is as follows:

```bash
jmspring@kudzu:~$ az account show
{
  "environmentName": "AzureCloud",
  "id": "7060bca0-1234-5-b54c-ab145dfaccef",
  "isDefault": true,
  "name": "jmspring trial account",
  "state": "Enabled",
  "tenantId": "72f984ed-86f1-41af-91ab-87acd01ed3ac",
  "user": {
    "name": "jmspring@kudzu.local",
    "type": "user"
  }
}
jmspring@kudzu:~$ az ad sp create-for-rbac --subscription "7060bca0-1234-5-b54c-ab145dfaccef"
{
  "appId": "7b6ab9ae-dead-abcd-8b52-0a8ecb5beef7",
  "displayName": "azure-cli-2019-06-13-04-47-36",
  "name": "http://azure-cli-2019-06-13-04-47-36",
  "password": "35591cab-13c9-4b42-8a83-59c8867bbdc2",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db47"
}
```

Take note of the following values, they will be needed for configuring Terraform as well as the deployment later:

- Subscription Id (`id` from account): `7060bca0-1234-5-b54c-ab145dfaccef`
- Tenant Id: `72f984ed-86f1-41af-91ab-87acd01ed3ac`
- Client Id (appId): `7b6ab9ae-dead-abcd-8b52-0a8ecb5beef7`
- Client Secret (password): `35591cab-13c9-4b42-8a83-59c8867bbdc2`

### Creating an RSA Key for Logging Into AKS Nodes

As above, we use `ssh-keygen` to generate another RSA keypair.  This key will be used by the Terraform scripts to setup the log in credentials on the nodes in the AKS cluster. We will use this key when going to setup the Terraform deployment variabled.

To generate the key:

```bash
kudzu:azure-simple jmspring$ ssh-keygen -b 4096 -t rsa -f ~/.ssh/azure-simple-node-key
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/jims/.ssh/azure-simple-node-key.
Your public key has been saved in /home/jims/.ssh/azure-simple-node-key.pub.
The key fingerprint is:
SHA256:+8pQ4MuQcf0oKT6LQkyoN6uswApLZQm1xXc+pp4ewvs jims@fubu
The key's randomart image is:
+---[RSA 4096]----+
|   ...           |
|  . o. o .       |
|.. .. + +        |
|... .= o *       |
|+  ++ + S o      |
|oo=..+ = .       |
|++ ooo=.o        |
|B... oo=..       |
|*+. ..oEo..      |
+----[SHA256]-----+
```

### Configure Terraform For Azure Access

Terraform supports a number of methods for authenticating with Azure.  The method Bedrock uses is [authenticating with a Service Principa and client secret](https://www.terraform.io/docs/providers/azurerm/auth/service_principal_client_secret.html).  This is done by setting a few environment variables via the Bash `export` command.  To set the variables, we will use the Service Principal defined [above](#creating-an-azure-service-rincipal).

Set the variables as follows:

```bash
kudzu:azure-simple jmspring$ export ARM_SUBSCRIPTION_ID=7060bca0-1234-5-b54c-ab145dfaccef
kudzu:azure-simple jmspring$ export ARM_TENANT_ID=72f984ed-86f1-41af-91ab-87acd01ed3ac
kudzu:azure-simple jmspring$ export kudzu:azure-simple jmspring$ ARM_CLIENT_SECRET=35591cab-13c9-4b42-8a83-59c8867bbdc2
kudzu:azure-simple jmspring$ export ARM_CLIENT_ID=7b6ab9ae-dead-abcd-8b52-0a8ecb5beef7
```

If you execute `env | grep ARM` you should see:

```bash
kudzu:azure-simple jmspring$ env | grep ARM
ARM_SUBSCRIPTION_ID=7060bca0-1234-5-b54c-ab145dfaccef
ARM_TENANT_ID=72f984ed-86f1-41af-91ab-87acd01ed3ac
ARM_CLIENT_SECRET=35591cab-13c9-4b42-8a83-59c8867bbdc2
ARM_CLIENT_ID=7b6ab9ae-dead-abcd-8b52-0a8ecb5beef7
```

### Clone The Bedrock Repository

The Bedrock repository is [here](https://github.com/microsoft/bedrock).  To clone it, simply use the `git` command line:

```bash
kudzu:azure-simple jmspring$ git clone https://github.com/microsoft/bedrock.git
Cloning into 'bedrock'...
remote: Enumerating objects: 37, done.
remote: Counting objects: 100% (37/37), done.
remote: Compressing objects: 100% (32/32), done.
remote: Total 2154 (delta 11), reused 11 (delta 5), pack-reused 2117
Receiving objects: 100% (2154/2154), 29.33 MiB | 6.15 MiB/s, done.
Resolving deltas: 100% (1022/1022), done.
```

To verify, let's navigate to the `bedrock/cluster/environments` directory and do an `ls`:

```bash
kudzu:environments jmspring$ ls -l
total 0
drwxr-xr-x   8 jmspring  staff  256 Jun 12 09:11 azure-common-infra
drwxr-xr-x  15 jmspring  staff  480 Jun 12 09:11 azure-multiple-clusters
drwxr-xr-x   6 jmspring  staff  192 Jun 12 09:11 azure-simple
drwxr-xr-x   7 jmspring  staff  224 Jun 12 09:11 azure-single-keyvault
drwxr-xr-x   7 jmspring  staff  224 Jun 12 09:11 azure-velero-restore
drwxr-xr-x   3 jmspring  staff   96 Jun 12 09:11 minikube
```

Each of the directories represent a common pattern supported within Bedrock.  You can read more about them [on the Bedrock github](https://github.com/microsoft/bedrock/tree/master/cluster/azure).  

### Setup Terraform Deployment Variables for Azure Simple

As mentioned, we will be using the `azure-simple`, changing into that directory and doing an `ls` reveals:

```bash
kudzu:environments jmspring$ cd azure-simple
kudzu:azure-simple jmspring$ ls -l
total 32
-rw-r--r--  1 jmspring  staff   460 Jun 12 09:11 README.md
-rw-r--r--  1 jmspring  staff  1992 Jun 12 09:11 main.tf
-rw-r--r--  1 jmspring  staff   703 Jun 12 09:11 terraform.tfvars
-rw-r--r--  1 jmspring  staff  2465 Jun 12 09:11 variables.tf
```

The inputs for a Terraform deployment are typically specified in a `.tfvars` file.  In the `azure-simple` repository, a skeleton exists in the form of `terraform.tfvars`.  The contents of `terraform.tfvars` is as follows:

```bash
kudzu:azure-simple jmspring$ cat terraform.tfvars
resource_group_name="resource-group-name"
resource_group_location="westus2"
cluster_name="cluster-name"
agent_vm_count = "3"
dns_prefix="dns-prefix"
service_principal_id = "client-id"
service_principal_secret = "client-secret"
ssh_public_key = "public-key"
gitops_ssh_url = "git@github.com:timfpark/fabrikate-cloud-native-manifests.git"
gitops_ssh_key = "<path to private gitops repo key>"
vnet_name = "<vnet name>"

#--------------------------------------------------------------
# Optional variables - Uncomment to use
#--------------------------------------------------------------
# gitops_url_branch = "release-123"
# gitops_poll_interval = "30s"
# gitops_path = "prod"
# network_policy = "calico"
```

At this point, we have values for `service_principal_id`, `service_principal_secret`, `ssh_public_key`, `gitops_ssh_url`, `gitops_ssh_key`.  For purposes of this walkthrough, `agent_vm_count` and `resource_group_location` are reasonable defaults.  So, let's define the remainder as follows:

- `resource_group_name`: `testazuresimplerg`
- `cluster_name`: `testazuresimplecluster`
- `dns_prefix`: `testazuresimple`
- `vnet_name`: `testazuresimplevnet`

`gitops_ssh_url` is the sample application repository that was previously [cloned](#forking-the-repository).  For this tutorial, given the GitHub user `jmspring`, the value is `git@github.com:jmspring/sample_app_manifests.git`.  It should also be noted, `gitops_ssh_key` is a *path* to the RSA private key we created [here](#create-an-rsa-key-pair-for-a-deploy-key-for-the-flux-repository) and `ssh_public_key` is the RSA public key that was created for [AKS node access](#creating-an-rsa-key-for-logging-into-aks-nodes).

Make a copy of the `terraform.tfvars` file and name it `testazuresimple.tfvars` for a working copy.  Next, using those values just defined and filling in the other values that were generated above, `testazuresimple.tfvars` should resemble:

```bash
kudzu:azure-simple jmspring$ cat testazuresimple.tfvars
resource_group_name="testazuresimplerg"
resource_group_location="westus2"
cluster_name="testazuresimplecluster"
agent_vm_count = "3"
dns_prefix="testazuresimple"
service_principal_id = "7b6ab9ae-dead-abcd-8b52-0a8ecb5beef7"
service_principal_secret = "35591cab-13c9-4b42-8a83-59c8867bbdc2"
ssh_public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCo5cFB/HiJB3P5s5kL3O24761oh8dVArCs9oMqdR09+hC5lD15H6neii4azByiMB1AnWQvVbk+i0uwBTl5riPgssj6vWY5sUS0HQsEAIRkzphik0ndS0+U8QI714mb3O0+qA4UYQrDQ3rq/Ak+mkfK0NQYL08Vo0vuE4PnmyDbcR8Pmo6xncj/nlWG8UzwjazpPCsP20p/egLldcvU59ikvY9+ZIsBdAGGZS29r39eoXzA4MKZZqXU/znttqa0Eed8a3pFWuE2UrntLPLrgg5hvliOmEfkUw0LQ3wid1+4H/ziCgPY6bhYJlMlf7WSCnBpgTq3tlgaaEHoE8gTjadKBk6bcrTaDZ5YANTEFAuuIooJgT+qlLrVql+QT2Qtln9CdMv98rP7yBiVVtQGcOJyQyG5D7z3lysKqCMjkMXOCH2UMJBrurBqxr6UDV3btQmlPOGI8PkgjP620dq35ZmDqBDfTLpsAW4s8o9zlT2jvCF7C1qhg81GuZ37Vop/TZDNShYIQF7ekc8IlhqBpbdhxWV6ap16paqNxsF+X4dPLW6AFVogkgNLJXiW+hcfG/lstKAPzXAVTy2vKh+29OsErIiL3SDqrXrNSmGmXwtFYGYg3XZLiEjleEzK54vYAbdEPElbNvOzvRCNdGkorw0611tpCntbpC79Q/+Ij6eyfQ== jmspring@kudzu"
gitops_ssh_url = "git@github.com:jmspring/sample_app_manifests.git"
gitops_ssh_key = "/home/jims/.ssh/azure-simple-deploy-key"
vnet_name = "testazuresimplevnet"

#--------------------------------------------------------------
# Optional variables - Uncomment to use
#--------------------------------------------------------------
# gitops_url_branch = "release-123"
# gitops_poll_interval = "30s"
gitops_path = "prod"
# network_policy = "calico"
```

Note, since our Flux deployment manifests are actually in a sub-directory within our [Flux Manifest Repository](#forking-the-repository), `gitops_path` was uncommented.

### Deploy the Azure Simple Template


### Interact with the Deployed Cluster