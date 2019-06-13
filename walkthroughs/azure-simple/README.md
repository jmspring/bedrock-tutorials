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

In addition, since the Flux manifest repository has a number of resource hungry components, we will need to modify `agent_vm_count` and add `agent_vm_size` as follows:

- `agent_vm_size` : `Standard_D4s_v3`
- `agent_vm_count` :`6`

`gitops_ssh_url` is the sample application repository that was previously [cloned](#forking-the-repository).  For this tutorial, given the GitHub user `jmspring`, the value is `git@github.com:jmspring/sample_app_manifests.git`.  It should also be noted, `gitops_ssh_key` is a *path* to the RSA private key we created [here](#create-an-rsa-key-pair-for-a-deploy-key-for-the-flux-repository) and `ssh_public_key` is the RSA public key that was created for [AKS node access](#creating-an-rsa-key-for-logging-into-aks-nodes).

Make a copy of the `terraform.tfvars` file and name it `testazuresimple.tfvars` for a working copy.  Next, using those values just defined and filling in the other values that were generated above, `testazuresimple.tfvars` should resemble:

```bash
kudzu:azure-simple jmspring$ cat testazuresimple.tfvars
resource_group_name="testazuresimplerg"
resource_group_location="westus2"
cluster_name="testazuresimplecluster"
agent_vm_size = "Standard_D4s_v3"
agent_vm_count = "6"
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

Note, since our Flux deployment manifests are actually in a sub-directory within our [Flux Manifest Repository](#forking-the-repository), `gitops_path` has been uncommented.

### Deploy the Azure Simple Template

With the Terraform variables file created, [testazuresimple.tfvars] (#setup-terraform-deployment-variables-for-azure-simple), it is time to do the Terraform deployment.  There are a couple of steps to this process:

- `terraform init` which initializes the local directory with metadata and other necessities Terraform needs.
- `terraform plan` which sanity checks your variables against the deployment
- `terraform apply` which actually deploys the infrastructure defined

Make sure one is in the `bedrock/cluster/environments/azure-simple` directory and that you know the path to `testazuresimple.tfvars` (it is assumed that is in the same directory as the `azure-simple` environment).

First execute `terraform init`:

```bash
kudzu:azure-simple jmspring$ terraform init
Initializing modules...
- module.provider
  Getting source "github.com/Microsoft/bedrock/cluster/azure/provider"
- module.vnet
  Getting source "github.com/Microsoft/bedrock/cluster/azure/vnet"
- module.aks-gitops
  Getting source "github.com/Microsoft/bedrock/cluster/azure/aks-gitops"
- module.provider.common-provider
  Getting source "../../common/provider"
- module.aks-gitops.aks
  Getting source "../../azure/aks"
- module.aks-gitops.flux
  Getting source "../../common/flux"
- module.aks-gitops.kubediff
  Getting source "../../common/kubediff"
- module.aks-gitops.aks.azure-provider
  Getting source "../provider"
- module.aks-gitops.aks.azure-provider.common-provider
  Getting source "../../common/provider"
- module.aks-gitops.flux.common-provider
  Getting source "../provider"
- module.aks-gitops.kubediff.common-provider
  Getting source "../provider"

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "null" (2.1.2)...
- Downloading plugin for provider "azurerm" (1.29.0)...
- Downloading plugin for provider "azuread" (0.3.1)...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Next, execute `terraform plan` and specify the location of our variables file:

```bash
kudzu:azure-simple jmspring$ terraform plan -var-file=testazuresimple.tfvars 
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + azurerm_resource_group.cluster_rg
      id:                                         <computed>
      location:                                   "westus2"
      name:                                       "testazuresimplerg"
      tags.%:                                     <computed>

  + module.vnet.azurerm_resource_group.vnet
      id:                                         <computed>
      location:                                   "westus2"
      name:                                       "testazuresimplerg"
      tags.%:                                     <computed>

  + module.vnet.azurerm_subnet.subnet
      id:                                         <computed>
      address_prefix:                             "10.10.1.0/24"
      ip_configurations.#:                        <computed>
      name:                                       "testazuresimplecluster-aks-subnet"
      resource_group_name:                        "testazuresimplerg"
      service_endpoints.#:                        "1"
      virtual_network_name:                       "testazuresimplevnet"

  + module.vnet.azurerm_virtual_network.vnet
      id:                                         <computed>
      address_space.#:                            "1"
      address_space.0:                            "10.10.0.0/16"
      location:                                   "westus2"
      name:                                       "testazuresimplevnet"
      resource_group_name:                        "testazuresimplerg"
      subnet.#:                                   <computed>
      tags.%:                                     "1"
      tags.environment:                           "azure-simple"

  + module.aks-gitops.module.aks.azurerm_kubernetes_cluster.cluster
      id:                                         <computed>
      addon_profile.#:                            <computed>
      agent_pool_profile.#:                       "1"
      agent_pool_profile.0.count:                 "6"
      agent_pool_profile.0.dns_prefix:            <computed>
      agent_pool_profile.0.fqdn:                  <computed>
      agent_pool_profile.0.max_pods:              <computed>
      agent_pool_profile.0.name:                  "default"
      agent_pool_profile.0.os_disk_size_gb:       "30"
      agent_pool_profile.0.os_type:               "Linux"
      agent_pool_profile.0.type:                  "AvailabilitySet"
      agent_pool_profile.0.vm_size:               "Standard_D4s_v3"
      agent_pool_profile.0.vnet_subnet_id:        "${var.vnet_subnet_id}"
      dns_prefix:                                 "testazuresimple"
      fqdn:                                       <computed>
      kube_admin_config.#:                        <computed>
      kube_admin_config_raw:                      <computed>
      kube_config.#:                              <computed>
      kube_config_raw:                            <computed>
      kubernetes_version:                         "1.13.5"
      linux_profile.#:                            "1"
      linux_profile.0.admin_username:             "k8sadmin"
      linux_profile.0.ssh_key.#:                  "1"
      linux_profile.0.ssh_key.0.key_data:         "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCo5cFB/HiJB3P5s5kL3O24761oh8dVArCs9oMqdR09+hC5lD15H6neii4azByiMB1AnWQvVbk+i0uwBTl5riPgssj6vWY5sUS0HQsEAIRkzphik0ndS0+U8QI714mb3O0+qA4UYQrDQ3rq/Ak+mkfK0NQYL08Vo0vuE4PnmyDbcR8Pmo6xncj/nlWG8UzwjazpPCsP20p/egLldcvU59ikvY9+ZIsBdAGGZS29r39eoXzA4MKZZqXU/znttqa0Eed8a3pFWuE2UrntLPLrgg5hvliOmEfkUw0LQ3wid1+4H/ziCgPY6bhYJlMlf7WSCnBpgTq3tlgaaEHoE8gTjadKBk6bcrTaDZ5YANTEFAuuIooJgT+qlLrVql+QT2Qtln9CdMv98rP7yBiVVtQGcOJyQyG5D7z3lysKqCMjkMXOCH2UMJBrurBqxr6UDV3btQmlPOGI8PkgjP620dq35ZmDqBDfTLpsAW4s8o9zlT2jvCF7C1qhg81GuZ37Vop/TZDNShYIQF7ekc8IlhqBpbdhxWV6ap16paqNxsF+X4dPLW6AFVogkgNLJXiW+hcfG/lstKAPzXAVTy2vKh+29OsErIiL3SDqrXrNSmGmXwtFYGYg3XZLiEjleEzK54vYAbdEPElbNvOzvRCNdGkorw0611tpCntbpC79Q/+Ij6eyfQ== jims@fubu"
      location:                                   "westus2"
      name:                                       "testazuresimplecluster"
      network_profile.#:                          "1"
      network_profile.0.dns_service_ip:           "10.0.0.10"
      network_profile.0.docker_bridge_cidr:       "172.17.0.1/16"
      network_profile.0.network_plugin:           "azure"
      network_profile.0.network_policy:           "azure"
      network_profile.0.pod_cidr:                 <computed>
      network_profile.0.service_cidr:             "10.0.0.0/16"
      node_resource_group:                        <computed>
      resource_group_name:                        "testazuresimplerg"
      role_based_access_control.#:                "1"
      role_based_access_control.0.enabled:        "true"
      service_principal.#:                        "1"
      service_principal.3262013094.client_id:     "7b6ab9ae-7de4-4394-8b52-0a8ecb5d2bf7"
      service_principal.3262013094.client_secret: <sensitive>
      tags.%:                                     <computed>

  + module.aks-gitops.module.aks.azurerm_resource_group.cluster
      id:                                         <computed>
      location:                                   "westus2"
      name:                                       "testazuresimplerg"
      tags.%:                                     <computed>

  + module.aks-gitops.module.aks.null_resource.cluster_credentials
      id:                                         <computed>
      triggers.%:                                 "2"
      triggers.kubeconfig_recreate:               ""
      triggers.kubeconfig_to_disk:                "true"

  + module.aks-gitops.module.flux.null_resource.deploy_flux
      id:                                         <computed>
      triggers.%:                                 "2"
      triggers.enable_flux:                       "true"
      triggers.flux_recreate:                     ""


Plan: 8 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

As seen from the output, a number of objects have been defined for creation.

The final step is to issue `terraform apply` which also requires the file containing the variables we defined above.  The output for `terraform apply` is quite long, so the snippet below will only contain the beginning and the end, the full output is [here](./extras/terraform_apply_log.txt) (sensitive output has been removed).  Note the beginning looks similar to `terraform plan` and the output contains the status of deploying each component.  Based on dependencies, Terraform deploys components in the proper order derived from a dependency graph.

```bash
kudzu:azure-simple jmspring$ terraform apply -var-file=testazuresimple.tfvars 

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + azurerm_resource_group.cluster_rg
      id:                                         <computed>
      location:                                   "westus2"
      name:                                       "testazuresimplerg"
      tags.%:                                     <computed>

  + module.vnet.azurerm_resource_group.vnet
      id:                                         <computed>
      location:                                   "westus2"
      name:                                       "testazuresimplerg"
      tags.%:                                     <computed>

  + module.vnet.azurerm_subnet.subnet
      id:                                         <computed>
      address_prefix:                             "10.10.1.0/24"
      ip_configurations.#:                        <computed>
      name:                                       "testazuresimplecluster-aks-subnet"
      resource_group_name:                        "testazuresimplerg"
      service_endpoints.#:                        "1"
      virtual_network_name:                       "testazuresimplevnet"

  + module.vnet.azurerm_virtual_network.vnet
      id:                                         <computed>
      address_space.#:                            "1"
      address_space.0:                            "10.10.0.0/16"
      location:                                   "westus2"
      name:                                       "testazuresimplevnet"
      resource_group_name:                        "testazuresimplerg"
      subnet.#:                                   <computed>
      tags.%:                                     "1"
      tags.environment:                           "azure-simple"

  + module.aks-gitops.module.aks.azurerm_kubernetes_cluster.cluster
      id:                                         <computed>
      addon_profile.#:                            <computed>
      agent_pool_profile.#:                       "1"
      agent_pool_profile.0.count:                 "6"
      agent_pool_profile.0.dns_prefix:            <computed>
      agent_pool_profile.0.fqdn:                  <computed>
      agent_pool_profile.0.max_pods:              <computed>
      agent_pool_profile.0.name:                  "default"
      agent_pool_profile.0.os_disk_size_gb:       "30"
      agent_pool_profile.0.os_type:               "Linux"
      agent_pool_profile.0.type:                  "AvailabilitySet"
      agent_pool_profile.0.vm_size:               "Standard_D4s_v3"
      agent_pool_profile.0.vnet_subnet_id:        "${var.vnet_subnet_id}"
      dns_prefix:                                 "testazuresimple"
      fqdn:                                       <computed>
      kube_admin_config.#:                        <computed>
      kube_admin_config_raw:                      <computed>
      kube_config.#:                              <computed>
      kube_config_raw:                            <computed>
      kubernetes_version:                         "1.13.5"
      linux_profile.#:                            "1"
      linux_profile.0.admin_username:             "k8sadmin"
      linux_profile.0.ssh_key.#:                  "1"
      linux_profile.0.ssh_key.0.key_data:         "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCo5cFB/HiJB3P5s5kL3O24761oh8dVArCs9oMqdR09+hC5lD15H6neii4azByiMB1AnWQvVbk+i0uwBTl5riPgssj6vWY5sUS0HQsEAIRkzphik0ndS0+U8QI714mb3O0+qA4UYQrDQ3rq/Ak+mkfK0NQYL08Vo0vuE4PnmyDbcR8Pmo6xncj/nlWG8UzwjazpPCsP20p/egLldcvU59ikvY9+ZIsBdAGGZS29r39eoXzA4MKZZqXU/znttqa0Eed8a3pFWuE2UrntLPLrgg5hvliOmEfkUw0LQ3wid1+4H/ziCgPY6bhYJlMlf7WSCnBpgTq3tlgaaEHoE8gTjadKBk6bcrTaDZ5YANTEFAuuIooJgT+qlLrVql+QT2Qtln9CdMv98rP7yBiVVtQGcOJyQyG5D7z3lysKqCMjkMXOCH2UMJBrurBqxr6UDV3btQmlPOGI8PkgjP620dq35ZmDqBDfTLpsAW4s8o9zlT2jvCF7C1qhg81GuZ37Vop/TZDNShYIQF7ekc8IlhqBpbdhxWV6ap16paqNxsF+X4dPLW6AFVogkgNLJXiW+hcfG/lstKAPzXAVTy2vKh+29OsErIiL3SDqrXrNSmGmXwtFYGYg3XZLiEjleEzK54vYAbdEPElbNvOzvRCNdGkorw0611tpCntbpC79Q/+Ij6eyfQ== jims@fubu"
      location:                                   "westus2"
      name:                                       "testazuresimplecluster"
      network_profile.#:                          "1"
      network_profile.0.dns_service_ip:           "10.0.0.10"
      network_profile.0.docker_bridge_cidr:       "172.17.0.1/16"
      network_profile.0.network_plugin:           "azure"
      network_profile.0.network_policy:           "azure"
      network_profile.0.pod_cidr:                 <computed>
      network_profile.0.service_cidr:             "10.0.0.0/16"
      node_resource_group:                        <computed>
      resource_group_name:                        "testazuresimplerg"
      role_based_access_control.#:                "1"
      role_based_access_control.0.enabled:        "true"
      service_principal.#:                        "1"
      service_principal.3262013094.client_id:     "7b6ab9ae-7de4-4394-8b52-0a8ecb5d2bf7"
      service_principal.3262013094.client_secret: <sensitive>
      tags.%:                                     <computed>

  + module.aks-gitops.module.aks.azurerm_resource_group.cluster
      id:                                         <computed>
      location:                                   "westus2"
      name:                                       "testazuresimplerg"
      tags.%:                                     <computed>

  + module.aks-gitops.module.aks.null_resource.cluster_credentials
      id:                                         <computed>
      triggers.%:                                 "2"
      triggers.kubeconfig_recreate:               ""
      triggers.kubeconfig_to_disk:                "true"

  + module.aks-gitops.module.flux.null_resource.deploy_flux
      id:                                         <computed>
      triggers.%:                                 "2"
      triggers.enable_flux:                       "true"
      triggers.flux_recreate:                     ""


Plan: 8 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

module.vnet.azurerm_resource_group.vnet: Creating...
  location: "" => "westus2"
  name:     "" => "testazuresimplerg"
  tags.%:   "" => "<computed>"
azurerm_resource_group.cluster_rg: Creating...
  location: "" => "westus2"
  name:     "" => "testazuresimplerg"
  tags.%:   "" => "<computed>"
azurerm_resource_group.cluster_rg: Creation complete after 3s (ID: /subscriptions/7060bca0-7a3c-44bd-b54c-...acfac/resourceGroups/testazuresimplerg)
module.vnet.azurerm_resource_group.vnet: Creation complete after 3s (ID: /subscriptions/7060bca0-7a3c-44bd-b54c-...acfac/resourceGroups/testazuresimplerg)
module.aks-gitops.module.aks.azurerm_resource_group.cluster: Creating...
  location: "" => "westus2"
  name:     "" => "testazuresimplerg"
  tags.%:   "" => "<computed>"
module.vnet.azurerm_virtual_network.vnet: Creating...
  address_space.#:     "" => "1"
  address_space.0:     "" => "10.10.0.0/16"
  location:            "" => "westus2"
  name:                "" => "testazuresimplevnet"
  resource_group_name: "" => "testazuresimplerg"
  subnet.#:            "" => "<computed>"
  tags.%:              "" => "1"
  tags.environment:    "" => "azure-simple"
module.aks-gitops.module.aks.azurerm_resource_group.cluster: Creation complete after 0s (ID: /subscriptions/7060bca0-7a3c-44bd-b54c-...acfac/resourceGroups/testazuresimplerg)
module.vnet.azurerm_virtual_network.vnet: Still creating... (10s elapsed)
module.vnet.azurerm_virtual_network.vnet: Creation complete after 14s (ID: /subscriptions/7060bca0-7a3c-44bd-b54c-...rk/virtualNetworks/testazuresimplevnet)
module.vnet.azurerm_subnet.subnet: Creating...
...
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): creating kubernetes namespace flux if needed
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): namespace/flux created
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): creating kubernetes secret flux-ssh from key file path /home/jims/.ssh/azure-simple-deploy-key
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): secret/flux-ssh created
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): Applying flux deployment
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): deployment.apps/flux created
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): configmap/flux-kube-config created
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): deployment.apps/flux-memcached created
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): service/flux-memcached created
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): clusterrole.rbac.authorization.k8s.io/flux created
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): clusterrolebinding.rbac.authorization.k8s.io/flux created
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): service/flux created
module.aks-gitops.module.flux.null_resource.deploy_flux (local-exec): serviceaccount/flux created
module.aks-gitops.module.flux.null_resource.deploy_flux: Creation complete after 8s (ID: 541853462335433160)
```

After `terraform apply` finishes, there is one critical output artifact.  In the `output` directory that was generated is the Kubernetes config file for the deployed cluster.  The default file is `output/bedrock_kube_config`.  This file will be needed for the following steps.

### Interact with the Deployed Cluster

Upon the deployment of the Bedrock AKS cluster, it is possible to start interacting with it.  Using the config file `output/bedrock_kube_config`, one of the first things we can do is list all pods deployed within the cluster:

```bash
KUBECONFIG=./output/bedrock_kube_config kubectl get po --all-namespaces
NAMESPACE       NAME                                            READY   STATUS                       RESTARTS   AGE
default         api-deploy-canary-5874f98bf8-4w4dh              0/1     CreateContainerConfigError   0          43m
default         api-deploy-canary-5874f98bf8-fgm2h              0/1     CreateContainerConfigError   0          43m
default         api-deploy-canary-5874f98bf8-ssft2              0/1     CreateContainerConfigError   0          43m
default         api-deploy-stable-647455778-hwgw4               0/1     CreateContainerConfigError   0          43m
default         api-deploy-stable-647455778-ld6fx               0/1     CreateContainerConfigError   0          43m
default         api-deploy-stable-647455778-pjfrs               0/1     CreateContainerConfigError   0          43m
default         spring-boot-ui-84f97849c-8qqr9                  2/2     Running                      0          40m
default         spring-boot-ui-84f97849c-cbp2t                  2/2     Running                      0          40m
default         spring-boot-ui-84f97849c-s68jb                  2/2     Running                      0          40m
elasticsearch   elasticsearch-client-6cb57d4664-q4msw           1/1     Running                      0          43m
elasticsearch   elasticsearch-client-6cb57d4664-w98sz           1/1     Running                      0          43m
elasticsearch   elasticsearch-data-0                            1/1     Running                      0          43m
elasticsearch   elasticsearch-data-1                            1/1     Running                      0          37m
elasticsearch   elasticsearch-master-0                          1/1     Running                      0          43m
elasticsearch   elasticsearch-master-1                          1/1     Running                      0          39m
elasticsearch   elasticsearch-master-2                          1/1     Running                      0          38m
elasticsearch   elasticsearch-test                              0/1     Error                        0          43m
fluentd         fluentd-elasticsearch-2dz4t                     1/1     Running                      0          43m
fluentd         fluentd-elasticsearch-7gkzl                     1/1     Running                      0          43m
fluentd         fluentd-elasticsearch-b2mjt                     1/1     Running                      0          43m
fluentd         fluentd-elasticsearch-mxwbs                     1/1     Running                      0          43m
fluentd         fluentd-elasticsearch-pbvvw                     1/1     Running                      0          43m
fluentd         fluentd-elasticsearch-rnx9t                     1/1     Running                      0          43m
flux            flux-7fb6b99bb4-l8xtb                           1/1     Running                      0          44m
flux            flux-memcached-757756884-rkwnv                  1/1     Running                      0          44m
grafana         grafana-588659577-74c9x                         1/1     Running                      0          43m
grafana         grafana-test                                    0/1     Error                        0          43m
istio-system    istio-citadel-7579f8fbb9-qd4k7                  1/1     Running                      0          43m
istio-system    istio-cleanup-secrets-1.1.2-gzgnm               0/1     Completed                    0          43m
istio-system    istio-galley-79d4c5d9f7-s26kw                   1/1     Running                      0          43m
istio-system    istio-ingressgateway-5fbcf4488f-2q2j2           1/1     Running                      0          43m
istio-system    istio-init-crd-10-x568m                         0/1     Completed                    0          43m
istio-system    istio-init-crd-11-4wv4c                         0/1     Completed                    0          43m
istio-system    istio-pilot-df78f86cb-rpvwq                     2/2     Running                      0          43m
istio-system    istio-policy-5df6f5457d-pw7g4                   2/2     Running                      0          43m
istio-system    istio-security-post-install-1.1.2-gkwx8         0/1     Completed                    2          43m
istio-system    istio-sidecar-injector-57f445c786-v5jb4         1/1     Running                      0          43m
istio-system    istio-telemetry-644b7d54c6-hj9np                2/2     Running                      4          43m
istio-system    kiali-5448cc49fb-cfn8h                          1/1     Running                      0          43m
jaeger          jaeger-agent-7gjxf                              1/1     Running                      0          43m
jaeger          jaeger-agent-kf6x7                              1/1     Running                      0          43m
jaeger          jaeger-agent-l9vgk                              1/1     Running                      0          43m
jaeger          jaeger-agent-m4bg7                              1/1     Running                      0          43m
jaeger          jaeger-agent-vkm9z                              1/1     Running                      0          43m
jaeger          jaeger-agent-wb4gm                              1/1     Running                      0          43m
jaeger          jaeger-collector-6557764f56-2cb8x               1/1     Running                      6          43m
jaeger          jaeger-collector-6557764f56-nnpz9               1/1     Running                      6          43m
jaeger          jaeger-collector-6557764f56-v22mf               1/1     Running                      0          43m
jaeger          jaeger-query-7bdc575489-5f8k4                   1/1     Running                      5          43m
jaeger          jaeger-query-7bdc575489-8lpcc                   1/1     Running                      5          43m
kibana          kibana-6dcc9c694f-9vgwv                         1/1     Running                      0          43m
kibana          kibana-test                                     0/1     Completed                    0          43m
kube-system     azure-cni-networkmonitor-868h9                  1/1     Running                      0          45m
kube-system     azure-cni-networkmonitor-8xxkh                  1/1     Running                      0          47m
kube-system     azure-cni-networkmonitor-9wn88                  1/1     Running                      0          47m
kube-system     azure-cni-networkmonitor-cpksc                  1/1     Running                      0          47m
kube-system     azure-cni-networkmonitor-mft8z                  1/1     Running                      0          47m
kube-system     azure-cni-networkmonitor-nlfjp                  1/1     Running                      0          47m
kube-system     azure-ip-masq-agent-5n48z                       1/1     Running                      0          47m
kube-system     azure-ip-masq-agent-5w47r                       1/1     Running                      0          47m
kube-system     azure-ip-masq-agent-cm8vl                       1/1     Running                      0          47m
kube-system     azure-ip-masq-agent-nhk2m                       1/1     Running                      0          47m
kube-system     azure-ip-masq-agent-nm5mx                       1/1     Running                      0          47m
kube-system     azure-ip-masq-agent-r8dm5                       1/1     Running                      0          45m
kube-system     azure-npm-5srf4                                 1/1     Running                      1          47m
kube-system     azure-npm-gbxfm                                 1/1     Running                      1          45m
kube-system     azure-npm-m4zcj                                 1/1     Running                      1          47m
kube-system     azure-npm-ngqpk                                 1/1     Running                      1          47m
kube-system     azure-npm-x24sc                                 1/1     Running                      1          47m
kube-system     azure-npm-zmw22                                 1/1     Running                      1          47m
kube-system     coredns-6b58b8549f-2d5sj                        1/1     Running                      0          46m
kube-system     coredns-6b58b8549f-rbktl                        1/1     Running                      0          50m
kube-system     coredns-autoscaler-7595c6bd66-lbmd7             1/1     Running                      0          50m
kube-system     kube-proxy-4zbmd                                1/1     Running                      0          47m
kube-system     kube-proxy-hq495                                1/1     Running                      0          47m
kube-system     kube-proxy-rt6lq                                1/1     Running                      0          45m
kube-system     kube-proxy-vxc6k                                1/1     Running                      0          47m
kube-system     kube-proxy-xggwj                                1/1     Running                      0          47m
kube-system     kube-proxy-xvdcq                                1/1     Running                      0          47m
kube-system     kubernetes-dashboard-69b6c88658-kk2zw           1/1     Running                      1          50m
kube-system     kured-gm478                                     1/1     Running                      0          43m
kube-system     kured-j99k6                                     1/1     Running                      0          43m
kube-system     kured-l8kvw                                     1/1     Running                      0          43m
kube-system     kured-n2mvr                                     1/1     Running                      0          43m
kube-system     kured-swhjk                                     1/1     Running                      0          43m
kube-system     kured-vpjqx                                     1/1     Running                      0          43m
kube-system     metrics-server-766dd9f7fd-pff2k                 1/1     Running                      1          50m
kube-system     tunnelfront-85845d7cc4-qkcdx                    1/1     Running                      0          50m
prometheus      prometheus-alertmanager-85b785566b-jpgt8        2/2     Running                      0          43m
prometheus      prometheus-kube-state-metrics-d46cdf5b4-zpklf   1/1     Running                      0          43m
prometheus      prometheus-node-exporter-52czp                  1/1     Running                      0          43m
prometheus      prometheus-node-exporter-5kkwh                  1/1     Running                      0          43m
prometheus      prometheus-node-exporter-brhkg                  1/1     Running                      0          43m
prometheus      prometheus-node-exporter-cjd88                  1/1     Running                      0          43m
prometheus      prometheus-node-exporter-hjdbn                  1/1     Running                      0          43m
prometheus      prometheus-node-exporter-sdrrt                  1/1     Running                      0          43m
prometheus      prometheus-pushgateway-5c949bfd75-hpj5c         1/1     Running                      0          43m
prometheus      prometheus-server-6876b75fbd-9qjc8              2/2     Running                      0          43m
```

The sample Flux repository deployed a "cloud native" stack which includes Istio, Kibana, Elastic Search, Jaeger, Elastic Search and others, as can be seen by the namespaces.  

One should note that there is also a namespace `flux`.  As previously mentioned, Flux is managing the deployment of all of the resources into the cluster.  Taking a look at the description for the flux pod `flux-7fb6b99bb4-4xfmk`, we see the following:

```bash
kudzu:azure-simple jmspring$ KUBECONFIG=./output/bedrock_kube_config kubectl describe po/flux-7fb6b99bb4-l8xtb --namespace=flux
Name:               flux-7fb6b99bb4-l8xtb
Namespace:          flux
Priority:           0
PriorityClassName:  <none>
Node:               aks-default-30249513-2/10.10.1.159
Start Time:         Thu, 13 Jun 2019 18:38:26 +0000
Labels:             app=flux
                    pod-template-hash=7fb6b99bb4
                    release=flux
Annotations:        <none>
Status:             Running
IP:                 10.10.1.161
Controlled By:      ReplicaSet/flux-7fb6b99bb4
Containers:
  flux:
    Container ID:  docker://57ab8cc87619fbf0ab00fc622817de126f49deaf4026c0870ece2c263860cbba
    Image:         docker.io/weaveworks/flux:1.12.2
    Image ID:      docker-pullable://weaveworks/flux@sha256:368bc5b219feffb1fe00c73cd0f1be7754591f86e17f57bc20371ecba62f524f
    Port:          3030/TCP
    Host Port:     0/TCP
    Args:
      --ssh-keygen-dir=/var/fluxd/keygen
      --k8s-secret-name=flux-ssh
      --memcached-hostname=flux-memcached
      --memcached-service=
      --git-url=git@github.com:jmspring/sample_app_manifests.git
      --git-branch=master
      --git-path=prod
      --git-user=Weave Flux
      --git-email=support@weave.works
      --git-set-author=false
      --git-poll-interval=5m
      --git-timeout=20s
      --sync-interval=5m
      --git-ci-skip=false
      --registry-poll-interval=5m
      --registry-rps=200
      --registry-burst=125
      --registry-trace=false
    State:          Running
      Started:      Thu, 13 Jun 2019 18:38:59 +0000
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:     50m
      memory:  64Mi
    Environment:
      KUBECONFIG:  /root/.kubectl/config
    Mounts:
      /etc/fluxd/ssh from git-key (ro)
      /etc/kubernetes/azure.json from acr-credentials (ro)
      /root/.kubectl from kubedir (rw)
      /var/fluxd/keygen from git-keygen (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from flux-token-79l8k (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kubedir:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      flux-kube-config
    Optional:  false
  git-key:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  flux-ssh
    Optional:    false
  git-keygen:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  acr-credentials:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/kubernetes/azure.json
    HostPathType:  
  flux-token-79l8k:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  flux-token-79l8k
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From                             Message
  ----    ------     ----  ----                             -------
  Normal  Scheduled  49m   default-scheduler                Successfully assigned flux/flux-7fb6b99bb4-l8xtb to aks-default-30249513-2
  Normal  Pulling    49m   kubelet, aks-default-30249513-2  pulling image "docker.io/weaveworks/flux:1.12.2"
  Normal  Pulled     49m   kubelet, aks-default-30249513-2  Successfully pulled image "docker.io/weaveworks/flux:1.12.2"
  Normal  Created    49m   kubelet, aks-default-30249513-2  Created container
  Normal  Started    49m   kubelet, aks-default-30249513-2  Started container
```

What is more interesting is to take a look at the Flux logs, one can checkout the activities that `flux` is performing.  A fuller log can be found [here](./extras/flux_log.txt).  But a snippet:

```bash
KUBECONFIG=./output/bedrock_kube_config kubectl log po/flux-7fb6b99bb4-l8xtb --namespace=flux
log is DEPRECATED and will be removed in a future version. Use logs instead.
ts=2019-06-13T18:38:59.332349059Z caller=main.go:193 version=1.12.2
ts=2019-06-13T18:38:59.381113324Z caller=main.go:350 component=cluster identity=/etc/fluxd/ssh/identity
ts=2019-06-13T18:38:59.381157125Z caller=main.go:351 component=cluster identity.pub="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDTNdGpnmztWRa8RofHl8dIGyNkEayNR6d7p2JtJ7+zMj0HRUJRc+DWvBML4DvT29AumVEuz1bsVyVS2f611NBmXHHKkbzAZZzv9gt2uB5sjnmm7LAORJyoBEodR/T07hWr8MDzYrGo5fdTDVagpoHcEke6JT04AL21vysBgqfLrkrtcgaXsw8e3rkfbqGLbhb6o1muGdEyE+uci4hRVj+FGL9twh3Mb6+0uak/UsTFgfDi/oTXdXOFIitQ1o40Eip6P4xejEOuIye0cg7rfX461NmOP7HIEsUa+BwMExiXXsbxj6Z0TXG0qZaQXWjvZF+MfHx/J0Alb9kdO3pYx3rJbzmdNFwbWM4I/zN+ng4TFiHBWRxRFmqJmKZX6ggJvX/d3z0zvJnvSmOQz9TLOT4lqZ/M1sARtABPGwFLAvPHAkXYnex0v93HUrEi7g9EnM+4dsGU8/6gx0XZUdH17WZ1dbEP7VQwDPnWCaZ/aaG7BsoJj3VnDlFP0QytgVweWr0J1ToTRQQZDfWdeSBvoqq/t33yYhjNA82fs+bR/1MukN0dCWMi7MqIs2t3TKYW635E7VHp++G1DR6w6LoTu1alpAlB7d9qiq7o1c4N+gakXSUkkHL8OQbQBeLeTG1XtYa//A5gnAxLSzxAgBpVW15QywFgJlPk0HEVkOlVd4GzUw=="
ts=2019-06-13T18:38:59.381176226Z caller=main.go:352 component=cluster host=https://10.0.0.1:443 version=kubernetes-v1.13.5
ts=2019-06-13T18:38:59.381259128Z caller=main.go:364 component=cluster kubectl=/usr/local/bin/kubectl
ts=2019-06-13T18:38:59.382887867Z caller=main.go:375 component=cluster ping=true
ts=2019-06-13T18:38:59.386733959Z caller=main.go:508 url=git@github.com:jmspring/sample_app_manifests.git user="Weave Flux" email=support@weave.works signing-key= sync-tag=flux-sync notes-ref=flux set-author=false
ts=2019-06-13T18:38:59.38677396Z caller=main.go:565 upstream="no upstream URL given"
ts=2019-06-13T18:38:59.387296272Z caller=main.go:586 addr=:3030
ts=2019-06-13T18:38:59.387398875Z caller=loop.go:90 component=sync-loop err="git repo not ready: git repo has not been cloned yet"
ts=2019-06-13T18:38:59.387487077Z caller=images.go:18 component=sync-loop msg="polling images"
ts=2019-06-13T18:38:59.387695782Z caller=images.go:28 component=sync-loop msg="no automated workloads"
ts=2019-06-13T18:38:59.79475761Z caller=checkpoint.go:21 component=checkpoint msg="update available" latest=1.13.0 URL=https://github.com/weaveworks/flux/releases/tag/1.13.0
ts=2019-06-13T18:39:00.028346389Z caller=warming.go:198 component=warmer info="refreshing image" image=gcr.io/google-containers/ip-masq-agent-amd64 tag_count=12 to_update=12 of_which_refresh=0 of_which_missing=12
ts=2019-06-13T18:39:00.474934606Z caller=warming.go:206 component=warmer updated=gcr.io/google-containers/ip-masq-agent-amd64 successful=12 attempted=12
ts=2019-06-13T18:39:00.47510221Z caller=images.go:18 component=sync-loop msg="polling images"
ts=2019-06-13T18:39:00.475146211Z caller=images.go:28 component=sync-loop msg="no automated workloads"
ts=2019-06-13T18:39:01.631322518Z caller=warming.go:198 component=warmer info="refreshing image" image=containernetworking/azure-npm tag_count=79 to_update=79 of_which_refresh=0 of_which_missing=79
ts=2019-06-13T18:39:03.427823776Z caller=warming.go:206 component=warmer updated=containernetworking/azure-npm successful=79 attempted=79
```

The current Flux Manifest when deployed appears to have some issues in the deployment of the API application.  This document will be updated upon fixes to that.