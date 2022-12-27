# Onboarding guidance of AZAP with AzureML

This article provides guidance on how to onboard your AZAP machine to Azure Machine Learning.

## AZAP provision with AKS

AZAP (Azure on Autopilot) is a dataplane platform that abstracts the HW infra layer from Data Plane services, thereby offering the flexibility to run Azure Data Plane anywhere, which is the mission statement for this initiative.

Now you can provision the AZAP VM SKU on AKS by following the [instructions](https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster?tabs=azure-cli#create-a-kubernetes-cluster), which provides a managed Kubernetes cluster based on an AZAP machine.

## Best practices of AzureML model training on AZAP

To integrate your AZAP machine to AzureML workspace for model training workloads, you need to follow this best practices.

### Prerequisites

* An AKS cluster with AZAP VM SKU is up and running in Azure.
    * If you have not previously used cluster extensions, you need to [register the KubernetesConfiguration service provider](https://learn.microsoft.com/en-us/azure/aks/dapr#register-the-kubernetesconfiguration-service-provider).
* Deploy the AzureML extension to your AKS cluster, more details please refer to [Deploy AzureML extension to AKS cluster](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-deploy-kubernetes-extension).
* Attach the AKS cluster to your AzureML workspace, more details please refer to [Attach AKS cluster to AzureML workspace](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-attach-kubernetes-to-workspace).

### System reserved resources

When the memory of a node is almost occupied by a pod, the node may run to OOM(out of memory) then would enter the `NotReady` state, meaning it cannot be used to run pods. To avoid this risk, it's recommended to reserve some system memory resources for the nodes. 

On the AzureML extension side, in order to prevent the node from running to OOM due to the ML workload eating up all the memory, we will automatically reserve **5% of the system memory** for the node by running a daemonset on the node. 

### Instance types setup rules

For training job on AZAP machine, it's recommended to setup the instance type with **resource requst equal to resource limit**.

In the case where the training job needs the entire node resource or half of the node resource, you need to create the instance type according to the following resource allocation rules:

More details about how to create the instance type, please refer to [here](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-manage-kubernetes-instance-types).

### Expose the instance type and instance count as paramters

To make the instance type and instance count of your training job configurable, you need to expose the instance type and instance count as parameters in your component spec yaml.

### Set environment variables for LightGBM jobs

For training jobs using LightGBM, to avoid [performance issues](#whats-the-performance-issues-with-lightgbm), the multi-thread settings in the job container should align with the requested CPU cores of the instance type you used. 

For example, for a training job requires a resource of `32c 256g`, you should specify an instance type with 32 core CPU request and 256GB memory request. Then you need to add the `environment_variable:` section to your component spec yaml and specify the following environment variables as the CPU request of this instance type:

```yaml
environment_variables:
    "MKL_NUM_THREADS": "32"            
    "NUMEXPR_NUM_THREADS": "32"            
    "OMP_NUM_THREADS": "32"
```

More guidance on how to create and run machine learning pipelines using components with the Azure Machine Learning CLI, please refer to [here](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-create-component-pipelines-cli).

#### Use AMLARC_PRESTEP for automatically setting these environment variables

In addition to the LgihtGBM, other distributed algorithms frameworks you used may also need to configure the mutli-thread settings for better performance.

In addition, we provide a `AMLARC_PRESTEP` environment variable to automatically set these environment variables mentioned above.

To automatically set these environment variables mentioned above, you can use the `AMLARC_PRESTEP` environment variable.
which will run some scripts and AzureML-provided functions before the job running, to automatically setup the cores count based on the instance type you used.

```bash
"environmentVariables": {"AMLARC_RESTEP": "export MKL_NUM_THREADS=` get_cores ` NUMEXPR_NUM_THREADS=` get_cores ` OMP_NUM_THREADS=` get_cores `"}
```

You should use a appropriate instance type with sufficient resource required to meet your CPU cores needs.

> **<span style="color:orange">Notes**:</span> 
> 1. The `AMLARC_PRESTEP` environment variable is only supported on Linux jobs.
> 1. For other frameworks, you can also use the `AMLARC_PRESTEP` for these environment variables automatically setting. 
>    1. But you need to make sure the environment variables the framework using to specify CPU resources are properly exported and passed to the `get_cores` function.

### Base image for Linux jobs

For general Linux jobs on AZAP, the base image should be based on **Ubuntu 20.04+**.

## Frequently Asked Questions

### Why is there a performance gap in model training when using AZAP machine on AzureML?

#### The default Disk has poor IO performance

#### Known performance issues with LightGBM

### Why should we need system reserved resources?
