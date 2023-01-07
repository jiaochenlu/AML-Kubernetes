# AzAP support and migration (Internal)

This article outlines the changes required to migrate your training workloads against AzureMLARC compute on the top of AzAP capacity in AzureML.

## Background

The goal of this migration is to enable WebXT to run AzureML workloads using **AzAP capacity** to achieve reduced costs from the compute side, to align with WebXT cost optimization on Azure.

Currently in AzureML, it's supported connecting a on-premises Kubernetes cluster or a cloud provided Kubernetes cluster (such as AKS) to the workspace as a compute target for running your workloads through [AzureMLARC service](https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster?tabs=azure-cli#create-a-kubernetes-cluster).

So right now the solution is to use AzureMLARC service, to attach either an AKS cluster provisioning the AzAP capacity or on-premises Kubernetes cluster setup on an AzAP machine to the AzureML workspace, then run your workloads against AzureMLARC compute.

## Best practices 

To use AzureMLARC service on the top of AzAP capacity to run your AzureML workloads, and ensure consistent training performance as before, you need to follow the best practices below for some required changes.

### Prerequisites

* An AKS cluster provisioning the AzAP capacity is up and running in Azure.
    * If you have not previously used cluster extensions, you need to [register the KubernetesConfiguration service provider](https://learn.microsoft.com/en-us/azure/aks/dapr#register-the-kubernetesconfiguration-service-provider).
* Deploy the AzureML extension to your AKS cluster, more details please refer to [Deploy AzureML extension to AKS cluster](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-deploy-kubernetes-extension).
* Attach the AKS cluster to your AzureML workspace, more details please refer to [Attach AKS cluster to AzureML workspace](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-attach-kubernetes-to-workspace).

### System will reserve resources by default

When the memory of a node is almost occupied by a pod, the node may run to OOM(out of memory) then would enter the `NotReady` state, meaning it cannot be used to run pods. To avoid this risk, it's recommended to reserve some system memory resources for the nodes. 

On the AzureML extension side, in order to prevent the node from running to OOM due to the ML workload eating up all the memory, we will automatically reserve **5% of the system memory** for the node by running a daemonset pod on the node.

### Use instance types to allocate compute resource

The instance type is a pre-defined resource allocation rule for the node. For AzureMLArc compute, you need to specify the instance type to allocate the compute resource for your training job.

To simplify the usage, when installing the AzureML extension in the AzAP capacity cluster, a series of standardized instance types with different resource allocation will be automatically created based on the VM SKU for you to choose from.

>[!NOTE]
>
>1. The resource allocation mechanism of these managed instance types is follows the [instance type setup rule](#instance-types-setup-rules).
>2. However, you can also create custom instance types according to your other resource requirement, but you need to follow the rules below as well.

More details about how to create the instance type, please refer to [here](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-manage-kubernetes-instance-types).

#### Instance types setup rules

For training job, it's recommended to setup the instance type with **resource request equal to resource limit**.

In the case where the training job needs the entire node resource or half of the node resource, since some resource has been reserved by system, to avoid job creating fail due to insufficient resource, you need to create the instance type according to the following resource allocation rules:  

<waiting for rule design based on capacity\>

### Expose the instance type and instance count as parameters

To make the instance type and instance count of your training job configurable, you need to expose the instance type and instance count as parameters in your component spec yaml.

<waiting for design from webxt\>

### Set environment variables for LightGBM jobs

For training jobs using LightGBM, the multi-thread settings in the job container should align with the requested CPU cores of the instance type you used.

For example, for a training job requires a resource of `32c256g`, you should specify an instance type with 32 core CPU request and 256GB memory request. Then you need to add the `environment_variable:` section to your component spec yaml and specify the following environment variables as the CPU request of this instance type:

```yaml
environment_variables:
    "MKL_NUM_THREADS": "32"            
    "NUMEXPR_NUM_THREADS": "32"            
    "OMP_NUM_THREADS": "32"
```

More guidance on how to create and run machine learning pipelines using components with the Azure Machine Learning CLI, please refer to [here](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-create-component-pipelines-cli).

Belows are list of this environment variables setting examples of some specific job case:

- LightGBM job
- PRS job 
- FAISS-ANN job
- Sweep job

Reset environment variables as runsetting for SweepComponent in SDK v1.5

```python
component_instance.environment_variables = {
    "MKL_NUM_THREADS": "32",
    "NUMEXPR_NUM_THREADS": "32",
    "OMP_NUM_THREADS": "32"
}
```

For the Sweepstep, you should define the command().sweep, and set the environment variables as

>[!NOTE]
>
> If you use PyTorch DataLoader to load data, the value of parameter `num_workers` will represent the number of data loading processes that torch will create. So for the **FAISS-ANN** job, you should set OMP_NUM_THREADS = (CPU cores / PyTorch dataloader worker number).

#### Use AMLARC_PRESTEP for automatically setting these environment variables

In addition to the LightGBM, other distributed algorithms frameworks you used may also need to configure the multi-thread settings for better performance. We provide a `AMLARC_PRESTEP` environment variable to automatically set these environment variables for your training jobs.

The `AMLARC_PRESTEP` environment variable will run some scripts and AzureML-provided functions before the training job running. You can just add it in the `environment_variables` section, export all framework environment variables and assign the `get_cores` function to them.

```bash
"AMLARC_RESTEP": "export <framework_env_variable>=` get_cores `"
```

Take LightGBM as an example, you can use the following `AMLARC_PRESTEP` to automatically set the `MKL_NUM_THREADS`, `NUMEXPR_NUM_THREADS` and `OMP_NUM_THREADS` environment variables for your training jobs:

```bash
"environmentVariables": {"AMLARC_RESTEP": "export MKL_NUM_THREADS=` get_cores ` NUMEXPR_NUM_THREADS=` get_cores ` OMP_NUM_THREADS=` get_cores `"}
```

> **<span style="color:orange">Notes**:</span> 
> 1. The `AMLARC_PRESTEP` environment variable is only supported on Linux jobs.
> 1. You should use a appropriate instance type with sufficient resource required to meet your CPU cores needs.
> 1. In the case where using the `AMLARC_PRESTEP` for other framework, you need to make sure the multi-thread environment variables of the are properly exported and passed to the `get_cores` function.

### Base image for Linux jobs

For general Linux jobs using AzAP capacity, the base image should be based on **Ubuntu 20.04+**.

### Job retry debugging
    
If the training job pod running in the cluster was terminated due to the node running to OOM, the job will be automatically retried to another available node.

To further debug the root cause of the job try, you can get the job-node mapping information in the **amlarc_cr_bootstrap.log** under system_logs folder.

The host name of the node which the job pod is running on will be indicated in this log, for example:

```bash
++ echo 'Run on node: ask-agentpool-17631869-vmss0000
```

Then you can access the cluster to check about the node status.
