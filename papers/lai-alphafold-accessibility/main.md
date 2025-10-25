---
title: 'AlphaFold accessibility: an optimized open-source OOD app for Protein Structure Prediction'
abstract: |
  AlphaFold, developed by DeepMind, has transformed structural biology by achieving unprecedented accuracy in protein structure prediction resulting in a 2024 Nobel Prize in Chemistry. Released in 2021 as open-source software, AlphaFold 2 enabled researchers worldwide to explore protein folding mechanisms and accelerate biomedical discovery. In contrast, AlphaFold 3 (released in 2024) is not open-source and is primarily accessible through a web interface, limiting community-driven optimization and large-scale computational deployment. To improve accessibility and efficiency, we developed an open-source web-portal implementation of AlphaFold 2 & 3 that optimizes computational resource allocation by intelligently separating CPU and GPU phases within a single Open OnDemand (OOD) instance. This design minimizes idle GPU time, reducing overall computational costs and improving throughput. Benchmarking on three major HPC systems (NCSA Delta, Jetstream2, and ROAR) demonstrates significant gains in resource efficiency. The resulting OOD-based platform is available through Penn State’s portal, lowering barriers for researchers across diverse disciplines who seek to integrate AlphaFold into their workflows.
---

## Introduction

Predicting a protein’s three-dimensional structure directly from its amino acid sequence was a central challenge in structural biology for decades. Recent advances in machine learning and artificial intelligence achieved a breakthrough with AlphaFold, developed by Google DeepMind, which demonstrated unprecedented accuracy in protein structure prediction. This innovation has reshaped structural biology and accelerated biomedical discovery. This achievement was honored with the 2024 Nobel Prize in Chemistry, awarded jointly to David Baker (University of Washington) for computational protein design, and to Demis Hassabis and John M. Jumper of Google DeepMind for advances in protein structure prediction using AlphaFold. The most recent version, AlphaFold 3, is proprietary, limiting transparency and broader community engagement with a technology that could significantly reshape structural biology and biomedical research. We present here an open-source web-portal implementation of AlphaFold v2/v2 on the Open OnDemand (OOD) ecosystem. This works helps to ensure that AlphaFold v2/v3 is more available to researchers across diverse disciplines. We benchmarked currently available AlphaFold software and databases and optimized our implementation to allow the software to be available to a much wider community.

We also performed extensive benchmarking of our implementation to demonstrate its utility and accessibility as the foundational service in the OOD ecosystem. Currently AlphaFold 2/3 software at scale is dominantly available behind industry sponsored paywalls. Meanwhile research institutions are continually under pressure to optimize resources and minimize expenses. Our results and accompanying open-source code repositories share these workflow optimizations and user-friendly OOD interface to the scientific community. The optimizations and the benchmarking performed demonstrate an improved ability to offer this resource as a scalable service on local or national HPC systems currently available to researchers.

Our teams validated this approach across three major infrastructures: NCSA Delta, Indiana University’s Jetstream, and Penn State University's Roar. At Penn State we leverage these modules for undergraduate and graduate student instruction in biochemistry and bioinformatics courses through its integration with our Open OnDemand portal. Our bioinformatics students and faculty researchers appreciate the GUI access to the well-known software. This work demonstrates a model for accessible, scalable services as the need increases for user-friendly solutions for research disciplines that have not traditionally been trained on high performance systems. Additionally, as this platform code is freely available and was developed in collaboration with other national supercomputing centers, there is the ability to deploy AlphaFold as a supported service across the country.

## Objectives – Addressing the Challenges

The challenge for this work was to meet the objectives for optimizing computational workflows and enhancing accessibility while retaining open-source deployment resources for the academic community. Optimizing the computational workflows addressed several key issues identified in our testing of the software on various HPC systems including deployment processes and the high computational resource demands on CPU/GPU utilization and storage requirements.

The deployment process initially supported Docker-only solutions which is problematic for HPC systems. Docker installations typically require root access to manage the containers, which is seldom an option for HPC services at an institution or a national center. Docker installations are also not strictly isolated as needed on shared and distributed systems. Through extensive benchmarking across three major clusters (NCSA Delta, Jetstream, and Roar), we identified that AlphaFold's workflow can be effectively split into CPU-intensive (MSA generation) and GPU-intensive (structure prediction) phases. Our analysis revealed that approximately 75% of the runtime is CPU-bound, while GPU resources are only required for the final structure prediction phase. This meant that typical job requests on GPU systems would incur GPU idle time and waste resources while the system was reserved but not in use.

Also, the approximately 5TB+ database requirements would incur a large storage cost if repeated across individual researchers instead of leveraging single shared installations of required files. A single-source for database and training data files that is shared would need to include planning for regular updates and maintenance of reference data over time.

Other differences between systems that were benchmarked was the type of storage technology utilized by the different sites. VAST Data utilizes NFS over RDMA (Remote Direct Memory Access) running over InfiniBand to deliver high-performance data access for demanding workloads such as AI/ML training and large-scale HPC applications. This architecture takes advantage of RDMA’s low latency and high bandwidth to significantly improve storage throughput and reduce CPU overhead, compared to traditional NFS over TCP/IP.

Specific observations for each of the cluster platforms was noted [@tab:benchmarked-systems].

```{table} Benchmarked systems used for the optimizations.
:label: tab:benchmarked-systems

|  | NCSA Delta | Jetstream2 | PSU Roar |
| :---- | :---- | :---- | :---- |
| **GPU support** | MIG-enabled support | Virtual GPUs | MIG-enabled support |
| **Job Submission** | Parallel job submissions | Sequential job processing on GPU systems | Parallel job submissions |
| **Workload Distributions** | Queue-based workload  | Dedicated systems assignments | Queue-based workload |
| **Partitioning support** | Tested on partitions down to 1/7th A100. Degraded performance possible on partitioned A100 | Only supported to ½ A100 for vGPUs | Tested on partitions down to 1/7th A100. Degraded performance possible on partitioned A100 |
| **GPU system** | Dual Nvidia A100 |  | Dual Nvidia A100 |
| **Storage Systems** | Delta offers disk-based Lustre storage parallel file system. The tiered storage architecture and parallel file system uses flash-based storage for high-speed workloads, and local NVMe solid-state disks (SSDs) on each compute node. | Jetstream2 offers an object store that uses OpenStack Swift. It uses a Ceph-based storage system. | VAST Storage systems utilizes NFS over RDMA (Remote Direct Memory Access) running over InfiniBand |
```

## Results – Implementation

```{figure} images/mono-cpu-time
:label: fig:mono-cpu-time

CPU Execution Time for Monomers. Measured runtime (seconds) for constructing multiple sequence alignments (MSAs) with jackhmmer given native amino acid sequences on three platforms (Penn State Roar, NCSA Delta, Jetstream2). Results show that runtime depends strongly on sequence identity, not just sequence length or hardware.
```

```{figure} images/mono-random-cpu-time
:label: fig:mono-random-cpu-time

CPU Execution Time for randomized monomeric sequences in @fig:mono-cpu-time. Measured runtime (seconds) for the same sequences after random shuffling of residues (length preserved). Randomization drastically alters execution time, indicating that CPU cost of MSA construction is governed by sequence content i.e., through the number and strength of database hits rather than by query length alone.
```

```{figure} images/multi-cpu-time
:label: fig:multi-cpu-time

Performance Analysis – Multimers. CPU Execution Time for Multimers.
```

```{figure} images/mono-random-gpu-time
:label: fig:mono-random-gpu-time

GPU Execution Runtime (seconds) for the structure module. With randomized queries, GPU time is governed primarily by chain length L; longer monomers take longer across all systems, with hardware shifting absolute levels but not the trend.
```

```{figure} images/multi-gpu-time
:label: fig:multi-gpu-time

Performance Analysis – Multimers. GPU Execution Time for Multimers. Neural network inference time on GPU scales with total sequence length. Faster accelerators reduce the intercept, but all systems exhibit a roughly positive correlation between complex size and runtime.
```

```{figure} images/multi-gpu-usage
:label: fig:multi-gpu-usage

GPU Utilization Analysis – Multimers. GPU Usage Summary. GPU usage declines as total sequence length decreases, with large complexes saturating devices (~90%) and smaller inputs leaving substantial headroom. This indicates that utilization efficiency is strongly tied to the length of the sequence.
```

## Results – Cost Optimization Strategies

Reviewing the benchmarking analysis, the following system-level optimizations were pursued. The approach was to dynamically distribute resource allocation to optimize best performance for CPU and GPU processing times, optimizing for GPU scheduling. The significant storage requirement for the database was shared across jobs. This approach resulted in the cost-savings in terms of reduction of idle GPU time, more efficient storage utilization, and automated resource scaling. All these measures would be important considerations in an environment where the software would be made available as a service to thousands of researchers simultaneously. Specifically, separating the CPU analysis from the GPU analysis requirements results in a 75% reduction in GPU allocation time and more intelligent resource scheduling.

```{figure} images/alphafold-input-ui
:label: fig:alphafold-input-ui

AlphaFold2 User Interface - Open OnDemand simple input form in sequence.
```

The approach to develop and provide a customized Open OnDemand module interface also enhances the ability to provide the software with a simple user interface. The AlphaFold OOD User Interface has only a single input requirement and provides real-time progress and logs as the software runs on the hosting systems. This project provides the Open OnDemand v3 module for running protein structure prediction jobs in a public Git repo (https://github.com/EpiGenomicsCode/ProteinStructure-OOD). The application developed simplifies the process of submitting and monitoring AlphaFold jobs by providing a user-friendly interface and automated job management. The application uses Singularity containers for its execution.

The Open OnDemand application also provides useful and easy to read job monitoring and results. @fig:alphafold-output-ui shows an example of the applications giving progress information while the jobs are running on the system and log files after completion.

The benefits and outcomes of this approach for HPC Centers include reducing the operational costs of providing this software that is available for AlphaFold2, 3 and Boltz while providing the convenience of centralized compliance and management. For the researchers there are the obvious benefits of simplified access to AlphaFold and protein discovery software without the usual coding requirements.

```{figure} images/alphafold-output-ui
:label: fig:alphafold-output-ui

AlphaFold2 User Interface - application outputs.
```

## Conclusions

Access to AI-intensive software has been shared via resource-intensive codebases or with an emphasis on leveraging for-fee vendor infrastructure. This approach, unfortunately, excludes the next generation of researchers from fully utilizing the benefits of these breakthrough models. Installation of such systems has been a challenge to provide at scale as a service. Our work shows the benefits of our collaborative research to optimize the workflows and leverage OOD as a robust containerized system. This work could help make this and presumably other AI-intensive software more widely available particularly for HPC systems engineers and those who support and deploy Open OnDemand portals at their own institutions.
