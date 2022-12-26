# Onboarding guidance of AZAP with AzureML

This article provides guidance on how to onboard your AZAP machine to Azure Machine Learning.

## AZAP provision with AKS?

AZAP (Azure on Autopilot) is a dataplane platform that abstracts the HW infra layer from Data Plane services, thereby offering the flexibility to run Azure Data Plane anywhere, which is the mission statement for this initiative.

Now you can provision the AZAP VM SKU on AKS by following the [instructions](https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster?tabs=azure-cli#create-a-kubernetes-cluster), which provides a managed Kubernetes cluster based on an AZAP machine.

## Best practices of AzureML model training on AZAP

To integrate your AZAP machine to AzureML workspace for model training workloads, you need to follow this best practices.

### Prerequisites

* An AKS cluster with AZAP VM SKU is up and running in Azure.
    * If you have not previously used cluster extensions, you need to [register the KubernetesConfiguration service provider](https://learn.microsoft.com/en-us/azure/aks/dapr#register-the-kubernetesconfiguration-service-provider).
* Deploy the AzureML extension to your AKS cluster, more details please refer to [Deploy AzureML extension to AKS cluster](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-deploy-kubernetes-extension).
* Attach the AKS cluster to your AzureML workspace, more details please refer to [Attach AKS cluster to AzureML workspace](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-attach-kubernetes-to-workspace).

### System reserved resources (SOP)

When a node in your Kubernetes cluster shuts down or crashes, it enters the NotReady state, meaning it cannot be used to run pods. All stateful pods running on the node then become unavailable. Many reasons will cause the node to crash, such as the node runs out of memory, for example if the memory in your cluster is fully occupied, which will cause the following issues:

* Operating system **OOM** (Out of Memory).
* `node not ready` error.

To prevent these issues, it's recommended to reserve some system memory resources for the nodes. On the AzureML extension side, we will automatically reserve **5% of the system memory** for the node by running a daemonset on the node. You can also reserve more system resources with defining appropriate resource requests and limits in your custom instance types.

### Setup instance types for one/half resource utilization

request equal to limit

### Expose the instance type and instance count in your component

### Set environment variables for LightGBM jobs

For training jobs with LightGBM, to avoid the [performance issues](#whats-the-performance-issues-with-lightgbm), the multi-thread settings in the job container should align with the required CPU core of the instance type you used. For example, for a training job that requires resource of `CPU 32 cores` and `Memory 256 GB`, you need to add the `environment_variable:` section to your component spec yaml and specify the following environment variables as the required CPU cores count:

```yaml
environment_variables:
    "MKL_NUM_THREADS": "32"            
    "NUMEXPR_NUM_THREADS": "32"            
    "OMP_NUM_THREADS": "32"
```

More guidance on how to create and run machine learning pipelines using components with the Azure Machine Learning CLI, please refer to [here](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-create-component-pipelines-cli).

#### Use AMLARC_PRESTEP for automatically setting these environment variables

To automatically set these environment variables, you can add the `AMLARC_PRESTEP` environment variable, which will trigger to run some scripts and functions provided by AzureML before the job running to automatically setup the cores count based on the instance type you used.

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
