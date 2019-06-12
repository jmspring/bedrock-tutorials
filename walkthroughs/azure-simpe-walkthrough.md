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

1. Fork the repository
2. Create an RSA keypair for the Flux repository
3. Add the keypair to the repository

#### Forking the Repository

Go to 