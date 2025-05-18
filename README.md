# Long-Read Genomics on the OSPool

This tutorial will walk you through a complete long-read sequencing analysis workflow using Oxford Nanopore data on the OSPool high-throughput computing ecosystem. You'll learn how to:

* Basecall raw Nanopore reads using the latest GPU-accelerated Dorado basecaller
* Map your reads to a reference genome using Minimap2
* Call structural variants using Sniffles2
* Breakdown massive bioinformatics workflows into many independent smaller tasks
* Submit hundreds to thousands of jobs with a few simple commands
* Use the Open Science Data Federation (OSDF) to manage file transfer during job submission

All of these steps are distributed across hundreds (or thousands!) of jobs using the HTCondor workload manager and Apptainer containers to run your software reliably and reproducibly at scale. The tutorial is built around realistic genomics use cases and emphasizes performance, reproducibility, and portability. You'll work with real data and see how high-throughput computing (HTC) can accelerate your genomics workflows.

>[!NOTE]
>If you're brand new to running jobs on the OSPool, we recommend completing the HTCondor ["Hello World"](https://portal.osg-htc.org/documentation/htc_workloads/workload_planning/htcondor_job_submission/) exercise before diving into this tutorial.

**Let’s get started!**

Jump to...
<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Long-Read Genomics on the OSPool](#long-read-genomics-on-the-ospool)
   * [Tutorial Setup](#tutorial-setup)
      + [Assumptions](#assumptions)
      + [Materials](#materials)
   * [Mapping Sequencing Reads to Genome](#mapping-sequencing-reads-to-genome)
      + [Data Wrangling and Splitting Reads](#data-wrangling-and-splitting-reads-1)
      + [Running Minimap to Map Reads to the Reference Genome](#running-minimap-to-map-reads-to-the-reference-genome)
   * [Next Steps](#next-steps)
      + [Software](#software)
      + [Data](#data)
      + [GPUs](#gpus)
   * [Getting Help](#getting-help)

<!-- TOC end -->

## Tutorial Setup

### Assumptions

This tutorial assumes that you:

* Have basic command-line experience (e.g., navigating directories, using bash, editing text files).
* Have a working OSPool account and can log into an Access Point (e.g., ap40.uw.osg-htc.org).
* Are familiar with HTCondor job submission, including writing simple .sub files and tracking job status with condor_q.
* Understand the general workflow of long-read sequencing analysis: basecalling → mapping → variant calling.
* Have access to a machine with a GPU-enabled execution environment (provided automatically via the OSPool).
* Have sufficient disk quota and file permissions in your OSPool home and OSDF directories.

>[!TIP]
>You do not need to be a genomics expert to follow this tutorial. The commands and scripts are designed to be beginner-friendly and self-contained, while still reflecting real-world research workflows.

### Materials

To obtain a copy of the files used in this tutorial, you can

* Clone the repository, with 
  
  ```
  git clone https://github.com/osg-htc/tutorial-long-read-genomics
  ```

  or the equivalent for your device

* Download the zip file of the materials: 
  [download here](https://github.com/osg-htc/tutorial-long-read-genomics/archive/refs/heads/main.zip)

## Mapping Sequencing Reads to Genome

### Data Wrangling and Splitting Reads

To get ready for our mapping step, we need to prepare our freshly basecalled reads. You should have a directory with several BAM files, these BAM files need to be sorted and 

1. Generate a list of the BAM files Dorado created during basecalling. Save it as `listOfBasecallingBAMs` in your OSDF directory. 

    ```
   ls /ospool/ap40/data/<user.name>/basecalledBAMs/ > /ospool/ap40/data/<user.name>/listOfBasecallingBAMs
   ```

### Running Minimap to Map Reads to the Reference Genome

1. Indexing our reference genome - Generating `Celegans_ref.mmi`

   1.  Create `minimap2_index.sh` using either `vim` or `nano`
        ```
       #!/bin/bash
       minimap2 -x map-ont -d Celegans_ref.mmi Celegans_ref.fa
       ```
   2. Create `minimap2_index.sub` using either `vim` or `nano`
        ```
        +SingularityImage      = "osdf:///ospool/ap40/data/<user.name>/genomics_tutorial/minimap2.sif"
    
        executable		       = ../executables/minimap2_index.sh
        
        transfer_input_files   = osdf:///ospool/ap40/data/<user.name>/genomics_tutorial/Celegans_ref.fa
    
        transfer_output_files  = ./Celegans_ref.mmi
        output_destination	   = osdf:///ospool/ap40/data/<user.name>/genomics_tutorial/
        
        output                 = ./minimap2/logs/$(Cluster)_$(Process)_indexing_step1.out
        error                  = ./minimap2/logs/$(Cluster)_$(Process)_indexing_step1.err
        log                    = ./minimap2/logs/$(Cluster)_$(Process)_indexing_step1.log
        
        request_cpus           = 4
        request_disk           = 10 GB
        request_memory         = 24 GB 
        
        queue 1
       ```
   3. Submit your `minimap2_index.sub` job to the OSPool
       ```
      condor_submit minimap2_index.sub
      ```
> [!WARNING]  
> Index will take a few minutes to complete, **do not proceed until your indexing job is completed**

2. Map our basecalled reads to the reference _C. elegans_ indexed genome - `Celegans_ref.mmi`
    
   1.  Create `minimap2_mapping.sh` using either `vim` or `nano`
       ```
       #!/bin/bash
       # Use minimap2 to map the basecalled reads to the reference genome
        ./minimap2 -ax map-ont Celegans_ref.mmi "$1" > "mapped_${1}_reads_to_genome.sam"
       
       # Use samtools to sort our mapped reads BAM, required for downstream analysis
       samtools sort "mapped_${1}_reads_to_genome.sam" -o "mapped_${1}_reads_to_genome_sam_sorted.bam"
       ```
   2. Create `minimap2_mapping.sub` using either `vim` or `nano`
       ```
        +SingularityImage      = "osdf:///ospool/ap40/data/<user.name>/minimap2.sif"
    
        executable		       = ../executables/minimap2_index.sh
        arguments              = $(BAM_File)
        transfer_input_files   = osdf:///ospool/ap40/data/<user.name>/Celegans_ref.mmi, osdf:///ospool/ap40/data/<user.name>/basecalledBAMs/$(BAM_FILE)
    
        transfer_output_files  = ./mapped_$(BAM_FILE)_reads_to_genome_sam_sorted.bam
        transfer_output_remaps = "mapped_$(BAM_FILE)_reads_to_genome_sam_sorted.bam=/home/<user.name>/genomics_tutorial/MappedBAMs/mapped_$(BAM_FILE)_reads_to_genome_sam_sorted.bam
        
        output                 = ./minimap2/logs/$(Cluster)_$(Process)_mapping_$(BAM_FILE)_step2.out
        error                  = ./minimap2/logs/$(Cluster)_$(Process)_mapping_$(BAM_FILE)_step2.err
        log                    = ./minimap2/logs/$(Cluster)_$(Process)_mapping_$(BAM_FILE)_step2.log
        
        request_cpus           = 2
        request_disk           = 5 GB
        request_memory         = 10 GB 
        
        queue BAM_File from /home/<user.name>/genomics_tutorial/listOfBasecallingBAMs
       ```
    
        In this step, we **are not** transferring our outputs using the OSDF. The mapped/sorted BAM files are intermediate temporary files in our analysis and do not benefit from the aggressive caching of the OSDF. By default, HTCondor will transfer outputs to the directory where we submitted our job from. Since we want to transfer the sorted mapped BAMs to a specific directory, we can use the `transfer_output_remaps` attribute on our submission script. The syntax of this attribute is:
   
        ```transfer_output_remaps = "<file_on_execution_point>=<desired_path_to_file_on_access_point>``` 
    
   3. Submit your cluster of minimap2 jobs to the OSPool
   
      ```
      condor_submit minimap2_mapping.sub
      ```

## Next Steps

Now that you've completed the long-read genomics tutorial on the OSPool, you're ready to adapt these workflows for your own data and research questions. Here are some suggestions for what you can do next:

🧬 Apply the Workflow to Your Own Data
* Replace the tutorial datasets with your own POD5 files and reference genome.
* Modify the basecalling, mapping, and variant calling submit files to fit your data size, read type (e.g., simplex vs. duplex), and resource needs.

🧰 Customize or Extend the Workflow
* Incorporate quality control steps (e.g., filtering or read statistics) using FastQC.
* Use other mappers or variant callers, such as ngmlr, pbsv, or cuteSV.
* Add downstream tools for annotation, comparison, or visualization (e.g., IGV, bedtools, SURVIVOR).

📦 Create Your Own Containers
* Extend the Apptainer containers used here with additional tools, reference data, or dependencies.
* For help with this, see our [Containers Guide](https://portal.osg-htc.org/documentation/htc_workloads/using_software/containers/).

🚀 Run Larger Analyses
* Submit thousands of basecalling or alignment jobs across the OSPool.
* Explore data staging best practices using the OSDF for large-scale genomics workflows.
* Consider using workflow managers (e.g., [DAGman](https://portal.osg-htc.org/documentation/htc_workloads/automated_workflows/dagman-workflows/) or [Pegasus](https://portal.osg-htc.org/documentation/htc_workloads/automated_workflows/tutorial-pegasus/)) with HTCondor.

🧑‍💻 Get Help or Collaborate
* Reach out to [support@osg-htc.org](mailto:support@osg-htc.org) for one-on-one help with scaling your research.
* Attend office hours or training sessions—see the [OSPool Help Page](https://portal.osg-htc.org/documentation/support_and_training/support/getting-help-from-RCFs/) for details.

### Software

In this tutorial, we created several *starter* apptainer containers, including tools like: Dorado, SAMtools, Minimap, and Sniffles2. These containers can serve as a *jumping-off* for you if you need to install additional software for your workflows. 

Our recommendation for most users is to use "Apptainer" containers for deploying their software.
For instructions on how to build an Apptainer container, see our guide [Using Apptainer/Singularity Containers](https://portal.osg-htc.org/documentation/htc_workloads/using_software/containers-singularity/).
If you are familiar with Docker, or want to learn how to use Docker, see our guide [Using Docker Containers](https://portal.osg-htc.org/documentation/htc_workloads/using_software/containers-docker/).

This information can also be found in our guide [Using Software on the Open Science Pool](https://portal.osg-htc.org/documentation/htc_workloads/using_software/software-overview/).

### Data

The ecosystem for moving data to, from, and within the HTC system can be complex, especially if trying to work with large data (> gigabytes).
For guides on how data movement works on the HTC system, see our [Data Staging and Transfer to Jobs](https://portal.osg-htc.org/documentation/htc_workloads/managing_data/overview/) guides.

### GPUs

The OSPool has GPU nodes available for common use, like the ones used in this tutorial. If you would like to learn more about our GPU capacity, please visit our [GPU Guide on the OSPool Documentation Portal](https://portal.osg-htc.org/documentation/htc_workloads/specific_resource/gpu-jobs/).

## Getting Help

The OSPool Research Computing Facilitators are here to help researchers using the OSPool for their research. We provide a broad swath of research facilitation services, including:

* **Web guides**: [OSPool Guides](https://portal.osg-htc.org/documentation/) - instructions and how-tos for using the OSPool and OSDF.
* **Email support**: get help within 1-2 business days by emailing [support@osg-htc.org](mailto:support@osg-htc.org).
* **Virtual office hours**: live discussions with facilitators - see the [Email, Office Hours, and 1-1 Meetings](https://portal.osg-htc.org/documentation/support_and_training/support/getting-help-from-RCFs/) page for current schedule.
* **One-on-one meetings**: dedicated meetings to help new users, groups get started on the system; email [support@osg-htc.org](mailto:support@osg-htc.org) to request a meeting.

This information, and more, is provided in our [Get Help](https://portal.osg-htc.org/documentation/support_and_training/support/getting-help-from-RCFs/) page.