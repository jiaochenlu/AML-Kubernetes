# Onboarding guidance of AZAP with AzureML

## What is AZAP?

## AZAP machine provision in AKS

## Best pratices of AzureML training jobs on AZAP

### Environment variable for LightGBM

For training jobs with LightGBM, to avoid the [performance issues](#whats-the-performance-issues-with-lightgbm), the multi-thread settings in the job container should align with the required CPU core of the instance type you used. For example, for a `32 cores 256 GB` resource needed training job, you need to set the envrionment variables as:
```bash
"MKL_NUM_THREADS": "32"            
"NUMEXPR_NUM_THREADS": "32"            
"OMP_NUM_THREADS": "32"
```
#### Use AMLARC_PRESTEP for automatically setting these envrionment variables

To automatically set these environment variables, you can add the `AMLARC_PRESTEP` envrionment variable, which will trigger to run some scripts and functions provided by AzureML before the job running to automatically setup the cores count based on the instance type you used.

```bash
"environmentVariables": {"AMLARC_RESTEP": "export MKL_NUM_THREADS=` get_cores ` NUMEXPR_NUM_THREADS=` get_cores ` OMP_NUM_THREADS=` get_cores `"}
```
You should specify a proper instance type with sufficient resource required to meet your CPU cores needs.

### Base image for linux job

For general linux jobs in AZAP, the base image should be based on **Ubuntun 20.04+**.

### System reserved resources (SOP)
To prevent [cluster collapse](#why-should-we-need-system-reserved-resources), we recommend you to reserve sufficient system resources.

For example, if the memory in your cluster is fully occupied, which will cause the operating system **OOM** or the **node not ready** issue.

#### Instance types setup


## Frequently Asked Questions

### What's the performance challenges on data IO for Cosmos mount?
there might be some perf challenges on data io for Cosmos mount in azap, we are still testing Shangmin is working on this

### What's the performance issues with LightGBM?

### Why should we need system reserved resources?
