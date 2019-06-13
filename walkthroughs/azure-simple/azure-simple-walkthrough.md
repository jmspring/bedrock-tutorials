## A Walkthrough of Deploying Azure Simple Bedrock Environment

This document walks through the necessary steps to deploy an Bedrock deployment using the 
[Azure Simple](https://github.com/microsoft/bedrock/cluster/environments/azure-simple) environment.  
This document will not include the whole of the [gitops](https://github.com/microsoft/bedrock/gitops)
workflow.  Instead, this document assumes a pre-existing [Flux Manifest repository](https://github.com/microsoft/bedrock/tree/master/cluster/common/flux)
which will be cloned and set up for the needs of this walkthrough.

Prior to getting started with the deployment, there are a couple of required steps:

1. The required common tools (kubectl, helm, and terraform) need to be installed.  That process is talked about [here](https://github.com/microsoft/bedrock/tree/master/cluster).  
2. The [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) needs to be installed.
3. Cloning and setting up the Flux manifest repository
4. Creating an [Azure Service Principal](https://github.com/microsoft/bedrock/tree/master/cluster/azure/service-principal)

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

###  Forking the Repository

To fork the repository, click [here](https://github.com/andrebriggs/sample_app_manifests/tree/master/prod).  You should see:

![initial repository](https://raw.githubusercontent.com/jmspring/bedrock-tutorials/master/walkthroughs/azure-simple/images/initial_repository.jpg)

